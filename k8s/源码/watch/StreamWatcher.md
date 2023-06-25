# StreamWatcher
StreamWatcher将一个Decoder和Reporter封装为Watcher。

## 结构体
```go
type StreamWatcher struct {
	sync.Mutex
	source   Decoder
	reporter Reporter
	result   chan Event
	done     chan struct{}
}

type Decoder interface {
    // 阻塞方法，返回一个Event和对应的资源对象。再调用Close()后此方法应该返回error
	Decode() (action EventType, object runtime.Object, err error)

    // 关闭底层io.reader
	Close()
}

// Reporter hides the details of how an error is turned into a runtime.Object for
// reporting on a watch stream since this package may not import a higher level report.
type Reporter interface {
	// 将一个err转化为ErrStatus
	AsObject(err error) runtime.Object
}
```
1. sync.Mutex：Stop方法使用的锁
2. source：数据源，一般为一个HTTP请求
3. result：watch使用的channel
4. done：Stop信号channel

## 构造方法
```go
func NewStreamWatcher(d Decoder, r Reporter) *StreamWatcher {
	sw := &StreamWatcher{
		source:   d,
		reporter: r,
		// It's easy for a consumer to add buffering via an extra
		// goroutine/channel, but impossible for them to remove it,
		// so nonbuffered is better.
		result: make(chan Event),
		// If the watcher is externally stopped there is no receiver anymore
		// and the send operations on the result channel, especially the
		// error reporting might block forever.
		// Therefore a dedicated stop channel is used to resolve this blocking.
		done: make(chan struct{}),
	}
    // 开启receive循环
	go sw.receive()
	return sw
}
```

## receive
```go
func (sw *StreamWatcher) receive() {
	defer utilruntime.HandleCrash()
	defer close(sw.result)
	defer sw.Stop()
	for {
		// for循环调用Decode
		action, obj, err := sw.source.Decode()
		if err != nil {
			switch err {
			case io.EOF:
				// 正常结束
				// watch closed normally
			case io.ErrUnexpectedEOF:
				// 异常结束
				klog.V(1).Infof("Unexpected EOF during watch stream event decoding: %v", err)
			default:
				if net.IsProbableEOF(err) || net.IsTimeout(err) {
					// 解析失败或超时
					klog.V(5).Infof("Unable to decode an event from the watch stream: %v", err)
				} else {
					select {
					case <-sw.done:
					// 向channel中添加一个Error类型的Event
					case sw.result <- Event{
						Type:   Error,
						Object: sw.reporter.AsObject(fmt.Errorf("unable to decode an event from the watch stream: %v", err)),
					}:
					}
				}
			}
			return
		}
		select {
		case <-sw.done:
			return
		// 向channel中添加一个正常类型的Event
		case sw.result <- Event{
			Type:   action,
			Object: obj,
		}:
		}
	}
}
```

## Stop
```go
// vendor/k8s.io/apimachinery/pkg/watcher/streamwatcher.go
func (sw *StreamWatcher) Stop() {
	// Call Close() exactly once by locking and setting a flag.
	sw.Lock()
	defer sw.Unlock()
	// closing a closed channel always panics, therefore check before closing
	select {
	case <-sw.done:
	default:
		// 加锁后向结束信号channel发送信号，并通知source关闭
		close(sw.done)
		sw.source.Close()
	}
}
```