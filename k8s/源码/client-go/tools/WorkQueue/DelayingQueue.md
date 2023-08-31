# DelayingQueue
DelayingQueue拥有`AddAfter(item interface{}, duration time.Duration)`方法，可以在添加元素时指定延迟时间。

## 接口
```go
// vendor/k8s.io/client-go/util/workqueue/delaying_queue.go
type DelayingInterface interface {
	// FIFO的基本接口
	Interface
	// AddAfter adds an item to the workqueue after the indicated duration has passed
    // 在指定时间之后再将元素添加到队列
	AddAfter(item interface{}, duration time.Duration)
}
```

## 实现类
```go
// vendor/k8s.io/client-go/util/workqueue/delaying_queue.go
type delayingType struct {
    // 基础队列
	Interface

	// clock tracks time for delayed firing
	clock clock.Clock

	// stopCh lets us signal a shutdown to the waiting loop
	stopCh chan struct{}
	// stopOnce guarantees we only signal shutdown a single time
	stopOnce sync.Once

	// heartbeat ensures we wait no more than maxWait before firing
	heartbeat clock.Ticker

	// waitingForAddCh is a buffered channel that feeds waitingForAdd
    // 用于传递waitFor的channel
	waitingForAddCh chan *waitFor

	// metrics counts the number of retries
	metrics retryMetrics
}

// waitFor保存了元素及其应被加入队列的时间
type waitFor struct {
	data    t
	readyAt time.Time
	// index in the priority queue (heap)
	index int
}
```

## 构造函数
```go
// vendor/k8s.io/client-go/util/workqueue/delaying_queue.go
// 传入基础队列，Ticker
func newDelayingQueue(clock clock.WithTicker, q Interface, name string) *delayingType {
	ret := &delayingType{
		Interface:       q,
		clock:           clock,
		heartbeat:       clock.NewTicker(maxWait),
		stopCh:          make(chan struct{}),
        // watiForAdd channel的缓存区是1000
		waitingForAddCh: make(chan *waitFor, 1000),
		metrics:         newRetryMetrics(name),
	}

    // 启动等待循环
	go ret.waitingLoop()
	return ret
}
```

## AddAfter
```go
// vendor/k8s.io/client-go/util/workqueue/delaying_queue.go
func (q *delayingType) AddAfter(item interface{}, duration time.Duration) {
	// don't add if we're already shutting down
	if q.ShuttingDown() {
		return
	}

	q.metrics.retry()

	// immediately add things with no delay
    // 如果时间间隔小于等于0，立即添加到普通队列中
	if duration <= 0 {
		q.Add(item)
		return
	}

	select {
	case <-q.stopCh:
		// unblock if ShutDown() is called
    // 将其包装为watiFor对象，传入waitingForChannel中  readyAt是入队时间
	case q.waitingForAddCh <- &waitFor{data: item, readyAt: q.clock.Now().Add(duration)}:
	}
}
```

## waitingLoop
```go
// 保持运行直到队列关闭
// vendor/k8s.io/client-go/util/workqueue/delaying_queue.go
func (q *delayingType) waitingLoop() {
	defer utilruntime.HandleCrash()

	// Make a placeholder channel to use when there are no items in our list
	never := make(<-chan time.Time)

	// Make a timer that expires when the item at the head of the waiting queue is ready
	var nextReadyAtTimer clock.Timer

    // waitForPriorityQueue是[]*waitFor，实现了heap.Interface，并提供了元素的比较方法(通过元素的入队时间比较)，
	waitingForQueue := &waitForPriorityQueue{}
	heap.Init(waitingForQueue)

    // 元素 -> waitFor的映射，用于元素去重
	waitingEntryByData := map[t]*waitFor{}

    // 循环分为两步
    // 1. 如果waitingForQueue中有元素，则不停取出元素(由于元素是按入队顺序排序的，所以取出的元素即是时间最接近的元素)，并判断是否到入队时间，如果到了，放入队列
	// 2. 从waitingForAddCh取出元素，放入waitingForQueue
	for {
		if q.Interface.ShuttingDown() {
			return
		}

		now := q.clock.Now()

		// Add ready entries
		for waitingForQueue.Len() > 0 {
			// 遍历判断waitingForQueue中的元素是否到了入队时间
			entry := waitingForQueue.Peek().(*waitFor)
			// 遍历到第一个没到入队时间的元素，说明后面的元素都是没到入队时间的，跳出循环
			if entry.readyAt.After(now) {
				break
			}

			// 弹出该元素，添加到队列中
			entry = heap.Pop(waitingForQueue).(*waitFor)
			q.Add(entry.data)
			delete(waitingEntryByData, entry.data)
		}

		// Set up a wait for the first item's readyAt (if one exists)
		nextReadyAt := never
		if waitingForQueue.Len() > 0 {
			// 如果waitingForQueue中仍然有未到入队时间的元素，则产生一个定时器，定时到最近一次入队时间
			if nextReadyAtTimer != nil {
				nextReadyAtTimer.Stop()
			}
			entry := waitingForQueue.Peek().(*waitFor)
			nextReadyAtTimer = q.clock.NewTimer(entry.readyAt.Sub(now))
			nextReadyAt = nextReadyAtTimer.C()
		}

		select {
		case <-q.stopCh:
			return

		// 用于确定阻塞时间没超过10s
		case <-q.heartbeat.C():
			// continue the loop, which will add ready items

		// 该channel收到消息说明waitingForQueue有元素到了入队时间
		case <-nextReadyAt:
			// continue the loop, which will add ready items

		// q.waitingForAddCh收到消息说明外部调用Add方法来添加元素
		case waitEntry := <-q.waitingForAddCh:
			if waitEntry.readyAt.After(q.clock.Now()) {
				// 如果没到入队时间，将元素添加到waitingForQueue，如果元素重复则更新入队时间
				insert(waitingForQueue, waitingEntryByData, waitEntry)
			} else {
				q.Add(waitEntry.data)
			}

			// 循环，直到将q.waitingForAddCh中的消息接收完成
			drained := false
			for !drained {
				select {
				case waitEntry := <-q.waitingForAddCh:
					if waitEntry.readyAt.After(q.clock.Now()) {
						insert(waitingForQueue, waitingEntryByData, waitEntry)
					} else {
						q.Add(waitEntry.data)
					}
				default:
					drained = true
				}
			}
		}
	}
}

func insert(q *waitForPriorityQueue, knownEntries map[t]*waitFor, entry *waitFor) {
	// if the entry already exists, update the time only if it would cause the item to be queued sooner
	existing, exists := knownEntries[entry.data]
	if exists {
		// 如果已存在，更新waitingForQueue
		if existing.readyAt.After(entry.readyAt) {
			existing.readyAt = entry.readyAt
			heap.Fix(q, existing.index)
		}

		return
	}

	heap.Push(q, entry)
	knownEntries[entry.data] = entry
}
```