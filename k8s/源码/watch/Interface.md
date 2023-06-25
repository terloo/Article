# Interface

## 接口
```go
// vendor/k8s.io/apimachinery/pkg/watch/watch.go
// 该接口的实现类用于定义一个Watcher，监听并汇报变化
type Interface interface {
	// Stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
    // 停止监听
	Stop()

	// Returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, the implementation will close this channel and
	// release any resources used by the watch.
    // 获得一个只读channel，从channel中可以读取Event
	ResultChan() <-chan Event
}
```

## 实现类
1. emptyWatch：一个空的Wacher，返回的channel是已关闭的
2. FakeWatcher：用于测试的假Watcher，内部的channel没有缓冲区，可以调用对应的方法传入对应的事件。线程安全
3. RaceFreeFakeWatcher：用于测试的假Watcher，内部的channel有默认长度(100)的缓冲区，可以调用对应的方法传入对应的事件。线程安全
4. ProxyWatcher：用于将一个现有的channel包装为Watcher。线程安全