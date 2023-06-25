# EvnetRecorder

## 接口
```go
// vendor/k8s.io/client-go/tools/record/event.go
type EventRecorder interface {
	// Event constructs an event from the given information and puts it in the queue for sending.
	// 'object' is the object this event is about. Event will make a reference-- or you may also
	// pass a reference to the object directly.
	// 'type' of this event, and can be one of Normal, Warning. New types could be added in future
	// 'reason' is the reason this event is generated. 'reason' should be short and unique; it
	// should be in UpperCamelCase format (starting with a capital letter). "reason" will be used
	// to automate handling of events, so imagine people writing switch statements to handle them.
	// You want to make that easy.
	// 'message' is intended to be human readable.
	//
	// The resulting event will be created in the same namespace as the reference object.
	Event(object runtime.Object, eventtype, reason, message string)

	// Eventf is just like Event, but with Sprintf for the message field.
	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})

	// AnnotatedEventf is just like eventf, but with annotations attached
	AnnotatedEventf(object runtime.Object, annotations map[string]string, eventtype, reason, messageFmt string, args ...interface{})
}
```
1. Event：对刚发生的事件进行记录
2. Eventf：格式化记录消息message
3. AnnotatedEventf：与Eventf一样，可以附加annotations信息
4. eventtype:
   1. `EventTypeNormal string = "Normal"`：普通事件
   2. `EventTypeWarning string = "Warning"`：警告事件

## 实现类
```go
// vendor/k8s.io/clinet-go/tools/record/event.go
type recorderImpl struct {
	scheme *runtime.Scheme
	// 事件来源
	source v1.EventSource
    // *watch.Broadcaster是该EventRecorder的消费者
	*watch.Broadcaster
	clock clock.PassiveClock
}

// 接口的三个方法最终会调用此内部方法
func (recorder *recorderImpl) generateEvent(object runtime.Object, annotations map[string]string, eventtype, reason, message string) {
	ref, err := ref.GetReference(recorder.scheme, object)
	if err != nil {
		klog.Errorf("Could not construct reference to: '%#v' due to: '%v'. Will not report event: '%v' '%v' '%v'", object, err, eventtype, reason, message)
		return
	}

	if !util.ValidateEventType(eventtype) {
		klog.Errorf("Unsupported event type: '%v'", eventtype)
		return
	}

    // 构造一个事件
	event := recorder.makeEvent(ref, annotations, eventtype, reason, message)
	event.Source = recorder.source

    // 调用消费者的方法来消费，该操作是一个非阻塞的方法
	if sent := recorder.ActionOrDrop(watch.Added, event); !sent {
		klog.Errorf("unable to record event: too many queued events, dropped event %#v", event)
	}
}

// 构造事件
func (recorder *recorderImpl) makeEvent(ref *v1.ObjectReference, annotations map[string]string, eventtype, reason, message string) *v1.Event {
	t := metav1.Time{Time: recorder.clock.Now()}
	namespace := ref.Namespace
	if namespace == "" {
		namespace = metav1.NamespaceDefault
	}
	return &v1.Event{
		ObjectMeta: metav1.ObjectMeta{
			// 事件的名字为 <对象名字>.<纳秒时间戳的16进制格式>
			Name:        fmt.Sprintf("%v.%x", ref.Name, t.UnixNano()),
			Namespace:   namespace,
			Annotations: annotations,
		},
		InvolvedObject: *ref,
		Reason:         reason,
		Message:        message,
		FirstTimestamp: t,
		LastTimestamp:  t,
		Count:          1,
		Type:           eventtype,
	}
}
```