# broadcasterWatcher
broadcasterWatcher可以获取一个channel，从channel中读取分发的事件

## 接口
```go
// vendor/k8s.io/apimachinery/pkg/watch/watch.go
type Interface interface {
	// Stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// Returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, the implementation will close this channel and
	// release any resources used by the watch.
	ResultChan() <-chan Event
}
```
1. Stop：停止消费
2. ResultChan：获取消费channel

## 实现类
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
type broadcasterWatcher struct {
	result  chan Event
	stopped chan struct{}
	stop    sync.Once
	id      int64
	m       *Broadcaster
}
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
func (mw *broadcasterWatcher) ResultChan() <-chan Event {
	return mw.result
}

func (mw *broadcasterWatcher) Stop() {
    // 防止多次执行导致panic
	mw.stop.Do(func() {
		close(mw.stopped)
		mw.m.stopWatching(mw.id)
	})
}
```