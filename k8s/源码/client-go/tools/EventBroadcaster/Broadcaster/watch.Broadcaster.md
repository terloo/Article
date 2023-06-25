# watch.Broadcaster
Broadcaster是apimachinery库中的提供的基础事件分发器。  
这里的事件指的是k8s中资源对象的删改查而不是Event资源对象。

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
type Broadcaster struct {
    // 保存所有的watcher
	watchers     map[int64]*broadcasterWatcher
    // 下一个watcher所使用的id号
	nextWatcher  int64
    // 一个用于优雅关闭的waitGroup
	distributing sync.WaitGroup

    // 接收Event使用的channel
	incoming chan Event
	stopped  chan struct{}

	// How large to make watcher's channel.
    // watcher用的channel的大小
	watchQueueLength int
	// If one of the watch channels is full, don't wait for it to become empty.
	// Instead just deliver it to the watchers that do have space in their
	// channels and move on to the next event.
	// It's more fair to do this on a per-watcher basis than to do it on the
	// "incoming" channel, which would allow one slow watcher to prevent all
	// other watchers from getting new events.
    // 如果watcher的channel满了之后的处理行为
	fullChannelBehavior FullChannelBehavior
}
```

## 构造函数
两个构造函数唯一的区别在于incoming的channel缓冲区长度是否为默认值
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
func NewBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		// 默认缓冲区长度为25
		incoming:            make(chan Event, incomingQueueLength),
		stopped:             make(chan struct{}),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	// waitGroup加一，表示已经开始分发循环了
	m.distributing.Add(1)
	go m.loop()
	return m
}

func NewLongQueueBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		incoming:            make(chan Event, queueLength),
		stopped:             make(chan struct{}),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	m.distributing.Add(1)
	go m.loop()
	return m
}
```

## loop()  开始分发Event的循环
```go
// vendor/k8s.io/apimachinery/pkg/match/mux.go

// 除了删改查等事件，还有一种用于执行函数的内部事件
const internalRunFunctionMarker = "internal-do-function"

func (m *Broadcaster) loop() {
	// Deliberately not catching crashes here. Yes, bring down the process if there's a
	// bug in watch.Broadcaster.
	for event := range m.incoming {
		if event.Type == internalRunFunctionMarker {
			// 如果是内部执行函数事件，则执行函数，一般是添加一个watcher
			event.Object.(functionFakeRuntimeObject)()
			continue
		}
		// 分发事件
		m.distribute(event)
	}

	// incoming关闭后关闭所有watcher
	m.closeAll()
	m.distributing.Done()
}

// functionFakeRuntimeObject，将一个func伪装为Runtime.Object，使其可以传入incoming channel
type functionFakeRuntimeObject func()

func (obj functionFakeRuntimeObject) GetObjectKind() schema.ObjectKind {
	return schema.EmptyObjectKind
}
func (obj functionFakeRuntimeObject) DeepCopyObject() runtime.Object {
	if obj == nil {
		return nil
	}
	// funcs are immutable. Hence, just return the original func.
	return obj
}
```

## Watcher()  添加并获取一个watcher
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
func (m *Broadcaster) Watch() Interface {
	var w *broadcasterWatcher
	// 使用blockQueue在分发循环中插入一段添加watcher的逻辑
	m.blockQueue(func() {
		// 给wachter一个用于map存储的id
		id := m.nextWatcher
		m.nextWatcher++
		// 构造一个broadcasterWatcher
		w = &broadcasterWatcher{
			result:  make(chan Event, m.watchQueueLength),
			stopped: make(chan struct{}),
			id:      id,
			m:       m,
		}
		m.watchers[id] = w
	})
	if w == nil {
		// The panic here is to be consistent with the previous interface behavior
		// we are willing to re-evaluate in the future.
		panic("broadcaster already stopped")
	}
	return w
}
```

## blockQueue()   在分发循环中插入一段逻辑
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
func (m *Broadcaster) blockQueue(f func()) {
	select {
	case <-m.stopped:
		return
	default:
	}
	var wg sync.WaitGroup
	wg.Add(1)
	m.incoming <- Event{
		// 向incoming中插入一个类型为internalRunFunctionMarker的事件
		Type: internalRunFunctionMarker,
		Object: functionFakeRuntimeObject(func() {
			defer wg.Done()
			f()
		}),
	}
	// 阻塞，直到函数执行完毕
	wg.Wait()
}
```

## Action()   接受要分发的Evnet
```go
// vendor/k8s.io/apimachinery/pkg/watch/mux.go
func (m *Broadcaster) Action(action EventType, obj runtime.Object) {
	// 直接向incoming中传入一个Evnet，阻塞
	m.incoming <- Event{action, obj}
}

func (m *Broadcaster) ActionOrDrop(action EventType, obj runtime.Object) bool {
	// 非阻塞
	select {
	case m.incoming <- Event{action, obj}:
		return true
	default:
		return false
	}
}
```


## distribute()   分发事件到watcher
```go
// vendor/k8s.io/apimachinery/pkg/mux.go
func (m *Broadcaster) distribute(event Event) {
	if m.fullChannelBehavior == DropIfChannelFull {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			// watcher的channel满了之后直接将事件丢弃
			default: // Don't block if the event can't be queued.
			}
		}
	} else {
		for _, w := range m.watchers {
			select {
			case w.result <- event:
			case <-w.stopped:
			}
			// watcher的channel满了之后阻塞等待
		}
	}
}
```