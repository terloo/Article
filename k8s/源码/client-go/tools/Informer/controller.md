# controller
controller通过Reflect从apiServer中生产Delta元素，从DeltaFIFO中消费数据取出Delta元素。  
根据元素的类型，一方面通过Indexer更新本地缓存，一方面调用Processer来触发注册到Informer的事件方法。

## 接口
```go
// vendor/k8s.io/client-go/tools/cache/controller.go
type Controller interface {
	// Run does two things.  One is to construct and run a Reflector
	// to pump objects/notifications from the Config's ListerWatcher
	// to the Config's Queue and possibly invoke the occasional Resync
	// on that Queue.  The other is to repeatedly Pop from the Queue
	// and process with the Config's ProcessFunc.  Both of these
	// continue until `stopCh` is closed.
    // 运行controller，将会构造并运行一个Reflector，并轮询Queue传递给Config的ProcessFunc进行处理。
	Run(stopCh <-chan struct{})

	// HasSynced delegates to the Config's Queue
    // 队列是否初始化完成
	HasSynced() bool

	// LastSyncResourceVersion delegates to the Reflector when there
	// is one, otherwise returns the empty string
    // 最新次同步的资源版本
	LastSyncResourceVersion() string
}
```

## 结构体
```go
// vendor/k8s.io/client-go/tools/cache/controller.go
type controller struct {
	config         Config
	reflector      *Reflector
	reflectorMutex sync.RWMutex
	clock          clock.Clock
}
```

## Run
[工作流程](工作流程.md)
```go
// vendor/k8s.io/client-go/tools/cache/controller.go
func (c *controller) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
    // 收到关闭信息时关闭队列
	go func() {
		<-stopCh
		c.config.Queue.Close()
	}()
	// 新建一个Reflect
	r := NewReflector(
		c.config.ListerWatcher,
		c.config.ObjectType,
		c.config.Queue,
		c.config.FullResyncPeriod,
	)
	r.ShouldResync = c.config.ShouldResync
	r.WatchListPageSize = c.config.WatchListPageSize
	r.clock = c.clock
	if c.config.WatchErrorHandler != nil {
		r.watchErrorHandler = c.config.WatchErrorHandler
	}

	c.reflectorMutex.Lock()
	c.reflector = r
	c.reflectorMutex.Unlock()

	var wg wait.Group

	// 开始生产
	wg.StartWithChannel(stopCh, r.Run)

	// 开始消费
	wait.Until(c.processLoop, time.Second, stopCh)
	wg.Wait()
}
```