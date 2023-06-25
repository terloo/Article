# FIFO队列
最基本的队列，列入插入、获取元素、获取队列长度。  
队列中拥有queue切片、dirty集合、processing集合三种数据结构，保证了
1. 一个元素在未被处理时不会被重复添加
2. 一个元素只有在正式处理完毕之后才会被添加到队列末尾

## Interface
WorkQueue中的限速队列、延迟队列都基于名为Interface的接口实现
```go
// vendor/k8s.io/client-go/util/workqueue/queue.go
type Interface interface {
    // 向队列添加元素
	Add(item interface{})
    // 返回队列长度
	Len() int
    // 获取队列头部的一个元素
	Get() (item interface{}, shutdown bool)
    // 标记队列中该元素已被处理
	Done(item interface{})
    // 关闭队列
	ShutDown()
	ShutDownWithDrain()
    // 查询队列是否正在关闭
	ShuttingDown() bool
}
```

## FIFO结构体
```go
// vendor/k8s.io/client-go/util/workqueue/queue.go
type Type struct {
	// queue defines the order in which we will work on items. Every
	// element of queue should be in the dirty set and not in the
	// processing set.
	// 所有元素列表，具有顺序性
	queue []t

	// dirty defines all of the items that need to be processed.
	dirty set

	// Things that are currently being processed are in the processing set.
	// These things may be simultaneously in the dirty set. When we finish
	// processing something and remove it from this set, we'll check if
	// it's in the dirty set, and if so, add it to the queue.
	processing set

	cond *sync.Cond

	// 是否正在关闭
	shuttingDown bool
	drain        bool

	metrics queueMetrics

	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.WithTicker
}

type empty struct{}
type t interface{}
type set map[t]empty
```
1. queue：实际存储元素的切片，保证了元素的有序性
2. dirty：标记所有需要被处理的元素集合，即使用Get()返回的元素。保证了在正式处理某个元素之前，该元素不会被添加多次到queue中。
3. processing：标记所有正在被处理的元素集合

## 构造函数
```go
// vendor/k8s.io/client-go/util/workqueue/queue.go
func New() *Type {
	return NewNamed("")
}

func NewNamed(name string) *Type {
	rc := clock.RealClock{}
	return newQueue(
		rc,
		globalMetricsFactory.newQueueMetrics(name, rc),
		defaultUnfinishedWorkUpdatePeriod,
	)
}

func newQueue(c clock.WithTicker, metrics queueMetrics, updatePeriod time.Duration) *Type {
	t := &Type{
		clock:                      c,
		dirty:                      set{},
		processing:                 set{},
		cond:                       sync.NewCond(&sync.Mutex{}),
		metrics:                    metrics,
		unfinishedWorkUpdatePeriod: updatePeriod,
	}

	// Don't start the goroutine for a type of noMetrics so we don't consume
	// resources unnecessarily
	if _, ok := metrics.(noMetrics); !ok {
		go t.updateUnfinishedWorkLoop()
	}

	return t
}

const defaultUnfinishedWorkUpdatePeriod = 500 * time.Millisecond
```

## Add
```go
// vendor/k8s.io/client-go/util/workqueue/queue.go
func (q *Type) Add(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	// 如果队列正在关闭
	if q.shuttingDown {
		return
	}
	// 或者这个元素已经存在与dirty中，不添加
	// 保证元素在进入被处理状态前不会被重复添加
	if q.dirty.has(item) {
		return
	}

	q.metrics.add(item)

	// 添加到dirty中
	q.dirty.insert(item)

	// 如果这个元素正在被处理，不将其添加到queue中，防止相同的元素同时被处理
	if q.processing.has(item) {
		return
	}

	// 保证顺序
	q.queue = append(q.queue, item)
	// 通知消费
	q.cond.Signal()
}
```

## Get
```go
// vendor/k8s.io/client-go/util/workqueue/queue.go
func (q *Type) Get() (item interface{}, shutdown bool) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()
	// 如果没有元素，则等待
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
	if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}

	// 取出队列头
	item = q.queue[0]
	// The underlying array still exists and reference this object, so the object will not be garbage collected.
	q.queue[0] = nil
	q.queue = q.queue[1:]

	q.metrics.get(item)

	// 将元素从dirty移动到processing中
	q.processing.insert(item)
	q.dirty.delete(item)

	return item, false
}
```

## Done
```go
// vendor/k8s.io/client-go/util/workqueue/queue.go
func (q *Type) Done(item interface{}) {
	q.cond.L.Lock()
	defer q.cond.L.Unlock()

	q.metrics.done(item)

	// 将元素从processing中移除
	q.processing.delete(item)
	if q.dirty.has(item) {
		// 如果元素在处理状态中又被ADD进来，将其添加到queue中
		q.queue = append(q.queue, item)
		// 唤醒消费者
		q.cond.Signal()
	} else if q.processing.len() == 0 {
		q.cond.Signal()
	}
}
```