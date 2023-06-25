# sharedProcessor
sharedProcessor管理着一组processListener。  
调用其run方法时，sharedProcessor将会调用所有processListener的run方法
调用其distribute时，sharedProcessor将会调用所有processListener的add方法

## 结构体
```go
// k8s.io/client-go/tools/cache/shared_informer.go
type sharedProcessor struct {
	listenersStarted bool
	listenersLock    sync.RWMutex
	listeners        []*processorListener
	syncingListeners []*processorListener
	clock            clock.Clock
	wg               wait.Group
}
```
1. listenersStarted：该sharedProcessor是否启动
2. syncingListeners：所有正在接收同步事件的Listener
3. listeners：所有正在接收除了同步事件以外事件的Listener
4. clock：时钟，可用于测试
5. wg：apimachinery中的wait包，用于启动goroutine并等待goroutine完成

## 添加listener
```go
// k8s.io/client-go/tools/cache/shared_informer.go
// 向sharedProcessor中添加一个listener，如果该sharedProcessor已经启动，则调用listener的run和pop方法
func (p *sharedProcessor) addListener(listener *processorListener) {
	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()

	p.addListenerLocked(listener)
	// 如果该sharedProcessor已经启动
	if p.listenersStarted {
		// 启动listener的run和pop
		p.wg.Start(listener.run)
		p.wg.Start(listener.pop)
	}
} 

func (p *sharedProcessor) addListenerLocked(listener *processorListener) {
	p.listeners = append(p.listeners, listener)
	p.syncingListeners = append(p.syncingListeners, listener)
}
```

## 启动
```go
// k8s.io/client-go/tools/cache/shared_informer.go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
	func() {
		p.listenersLock.RLock()
		defer p.listenersLock.RUnlock()
		// 启动所有的listener
		for _, listener := range p.listeners {
			p.wg.Start(listener.run)
			p.wg.Start(listener.pop)
		}
		p.listenersStarted = true
	}()
	// 等待停止信号
	<-stopCh
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()
	for _, listener := range p.listeners {
		// 关闭listener的渠道
		close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
	}
	// 等待listener停止pop和run
	p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```

## 判断该sharedProcessor是否需要推动同步事件
```go
// k8s.io/client-go/tools/cache/shared_informer.go
func (p *sharedProcessor) shouldResync() bool {
	p.listenersLock.Lock()
	defer p.listenersLock.Unlock()

	p.syncingListeners = []*processorListener{}

	resyncNeeded := false
	now := p.clock.Now()
	for _, listener := range p.listeners {
		// need to loop through all the listeners to see if they need to resync so we can prepare any
		// listeners that are going to be resyncing.
		// 遍历所有listner，判断其是否需要同步，如果需要，则将其放入processorListener切片中
		if listener.shouldResync(now) {
			resyncNeeded = true
			p.syncingListeners = append(p.syncingListeners, listener)
			listener.determineNextResync(now)
		}
	}
	return resyncNeeded
}
```

## 分发
```go
// k8s.io/client-go/tools/cache/shared_informer.go
func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	if sync {
		for _, listener := range p.syncingListeners {
			listener.add(obj)
		}
	} else {
		for _, listener := range p.listeners {
			listener.add(obj)
		}
	}
}
```

## 修改所有listener的同步周期
```go
// k8s.io/client-go/tools/cache/shared_informer.go
func (p *sharedProcessor) resyncCheckPeriodChanged(resyncCheckPeriod time.Duration) {
	p.listenersLock.RLock()
	defer p.listenersLock.RUnlock()

	for _, listener := range p.listeners {
		resyncPeriod := determineResyncPeriod(listener.requestedResyncPeriod, resyncCheckPeriod)
		listener.setResyncPeriod(resyncPeriod)
	}
}
```