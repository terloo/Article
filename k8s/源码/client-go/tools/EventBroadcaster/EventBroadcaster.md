# EventBroadcaster
EventBroadcaster是watch.Broadcaster的封装，专门用于处理Event资源对象

## 接口
```go
// vendor/k8s.io/client-go/tools/record/event.go
type EventBroadcaster interface {
	// StartEventWatcher starts sending events received from this EventBroadcaster to the given
	// event handler function. The return value can be ignored or used to stop recording, if
	// desired.
	StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface

	// StartRecordingToSink starts sending events received from this EventBroadcaster to the given
	// sink. The return value can be ignored or used to stop recording, if desired.
	StartRecordingToSink(sink EventSink) watch.Interface

	// StartLogging starts sending events received from this EventBroadcaster to the given logging
	// function. The return value can be ignored or used to stop recording, if desired.
	StartLogging(logf func(format string, args ...interface{})) watch.Interface

	// StartStructuredLogging starts sending events received from this EventBroadcaster to the structured
	// logging function. The return value can be ignored or used to stop recording, if desired.
	StartStructuredLogging(verbosity klog.Level) watch.Interface

	// NewRecorder returns an EventRecorder that can be used to send events to this EventBroadcaster
	// with the event source set to the given event source.
	NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder

	// Shutdown shuts down the broadcaster
	Shutdown()
}
```
1. StartEventWatcher：构造一个watcher，使用传入的evenHandler进行处理，返回该watcher
2. StartRecordingToSink：构造一个watcher，将事件交由指定的Sink进行处理(即调用apiserver保存到ETCD)
3. StartLogging：构造一个watcher，将事件交由指定的Logging进行处理(记录到日志)
4. StartStructuredLogging：构造一个watcher，将事件交由指定的结构化Logging进行处理
5. NewRecorder：获取一个新的事件记录器
6. Shutdown：关闭事件

## 实现类
```go
// vendor/k8s.io/client-go/tools/record/event.go
type eventBroadcasterImpl struct {
    // eventBroadcasterImpl实际上是apimachinery库中watch.Broadcaster的封装
	// 与分发有关的操作都是由watch.Broadcaster进行实现
	*watch.Broadcaster
	sleepDuration time.Duration
	options       CorrelatorOptions
}
```

## 构造方法
```go
// vendor/k8s.io/client-go/tools/record/event.go
// 构造eventBroadcasterImpl时会构造一个watch.Broadcaster
func NewBroadcaster() EventBroadcaster {
	return &eventBroadcasterImpl{
		// 构造一个watch.Broadcaster
		Broadcaster:   watch.NewLongQueueBroadcaster(maxQueuedEvents, watch.DropIfChannelFull),
		sleepDuration: defaultSleepDuration,  // 10秒，如果要与ETCD同步，指定同步间隔
	}
}

// 使用CorrelatorOptions来构造该eventBroadcasterImpl
func NewBroadcasterWithCorrelatorOptions(options CorrelatorOptions) EventBroadcaster {
	return &eventBroadcasterImpl{
		Broadcaster:   watch.NewLongQueueBroadcaster(maxQueuedEvents, watch.DropIfChannelFull),
		sleepDuration: defaultSleepDuration,
		options:       options,
	}
}
```

## StartEventWatcher
StartEventWatcher新建一个watcher处理指定自定义函数
```go
// vendor/k8s.io/client-go/tools/record/event.go
func (e *eventBroadcasterImpl) StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface {
	watcher := e.Watch()
	go func() {
		defer utilruntime.HandleCrash()
		for watchEvent := range watcher.ResultChan() {
			event, ok := watchEvent.Object.(*v1.Event)
			if !ok {
				// This is all local, so there's no reason this should
				// ever happen.
				continue
			}
			eventHandler(event)
		}
	}()
	return watcher
}
```

## StartRecordingToSink
StartRecordingToSink新建一个watcher将事件通过apiserver保存到ETCD  
EventSink可以通过`&typecorev1.EventSinkImpl{Interface: clientSet.CoreV1().Events("")}`来获取
```go
// vendor/k8s.io/client-go/tools/record/event.go
func (e *eventBroadcasterImpl) StartRecordingToSink(sink EventSink) watch.Interface {
	// 构造一个事件聚合器
	eventCorrelator := NewEventCorrelatorWithOptions(e.options)

	// 开始watch，函数为recordToSink，默认与ETCD同步的事件为10s
	return e.StartEventWatcher(
		func(event *v1.Event) {
			recordToSink(sink, event, eventCorrelator, e.sleepDuration)
		})
}

func recordToSink(sink EventSink, event *v1.Event, eventCorrelator *EventCorrelator, sleepDuration time.Duration) {
	// Make a copy before modification, because there could be multiple listeners.
	// Events are safe to copy like this.
	eventCopy := *event
	event = &eventCopy
	// 聚合器会缓存最近发送给sink的事件，如果同一个事件重复发生，聚合器会直接将事件的发生数+1
	result, err := eventCorrelator.EventCorrelate(event)
	if err != nil {
		utilruntime.HandleError(err)
	}
	if result.Skip {
		return
	}
	tries := 0
	for {
		// 记录事件，Count大于1的事件会被更新
		if recordEvent(sink, result.Event, result.Patch, result.Event.Count > 1, eventCorrelator) {
			break
		}
		tries++
		if tries >= maxTriesPerEvent {
			klog.Errorf("Unable to write event '%#v' (retry limit exceeded!)", event)
			break
		}
		// Randomize the first sleep so that various clients won't all be
		// synced up if the master goes down.
		if tries == 1 { // 第一次同步要把时间错开
			time.Sleep(time.Duration(float64(sleepDuration) * rand.Float64()))
		} else {
			time.Sleep(sleepDuration)
		}
	}
}

func recordEvent(sink EventSink, event *v1.Event, patch []byte, updateExistingEvent bool, eventCorrelator *EventCorrelator) bool {
	var newEvent *v1.Event
	var err error
	if updateExistingEvent {
		// Count大于1的事件会被更新而不是创建
		newEvent, err = sink.Patch(event, patch)
	}
	// Update can fail because the event may have been removed and it no longer exists.
	if !updateExistingEvent || (updateExistingEvent && util.IsKeyNotFoundError(err)) {
		// Making sure that ResourceVersion is empty on creation
		event.ResourceVersion = ""
		// 如果不是更新已有Event，则创建事件
		newEvent, err = sink.Create(event)
	}
	if err == nil {
		// we need to update our event correlator with the server returned state to handle name/resourceversion
		// 更新聚合器缓存
		eventCorrelator.UpdateState(newEvent)
		return true
	}

	// If we can't contact the server, then hold everything while we keep trying.
	// Otherwise, something about the event is malformed and we should abandon it.
	// 判断error的类型进行处理
	switch err.(type) {
	case *restclient.RequestConstructionError:
		// We will construct the request the same next time, so don't keep trying.
		klog.Errorf("Unable to construct event '%#v': '%v' (will not retry!)", event, err)
		return true
	case *errors.StatusError:
		if errors.IsAlreadyExists(err) {
			klog.V(5).Infof("Server rejected event '%#v': '%v' (will not retry!)", event, err)
		} else {
			klog.Errorf("Server rejected event '%#v': '%v' (will not retry!)", event, err)
		}
		return true
	case *errors.UnexpectedObjectError:
		// We don't expect this; it implies the server's response didn't match a
		// known pattern. Go ahead and retry.
	default:
		// This case includes actual http transport errors. Go ahead and retry.
	}
	klog.Errorf("Unable to write event: '%#v': '%v'(may retry after sleeping)", event, err)
	return false
}
```

## NewRecorder
```go
// vendor/k8s.io/client-go/tools/record/event.go
func (e *eventBroadcasterImpl) NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder {
	// 构造一个recorderImpl
	return &recorderImpl{scheme, source, e.Broadcaster, clock.RealClock{}}
}
```
