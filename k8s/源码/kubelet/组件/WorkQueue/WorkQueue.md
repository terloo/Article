# WorkQueue
WorkQueue是kubelet使用的在每个元素上附有到期时间的队列

## 接口
```go
// kubelet/util/queue/work_queue.go
type WorkQueue interface {
	// GetWork dequeues and returns all ready items.
	GetWork() []types.UID
	// Enqueue inserts a new item or overwrites an existing item.
	Enqueue(item types.UID, delay time.Duration)
}
```
1. GetWork：弹出队列里所有当前时间大于到期时间的元素
2. Enqueue：插入一个新元素，并指定该元素的到期时间

## 实现类
```go
// kubelet/util/queue/work_queue.go
type basicWorkQueue struct {
	clock clock.Clock
	lock  sync.Mutex
	queue map[types.UID]time.Time
}
```

## 构造函数
```go
// kubelet/util/queue/work_queue.go
func NewBasicWorkQueue(clock clock.Clock) WorkQueue {
	queue := make(map[types.UID]time.Time)
	return &basicWorkQueue{queue: queue, clock: clock}
}
```

## 方法
```go
// kubelet/util/queue/work_queue.go
func (q *basicWorkQueue) GetWork() []types.UID {
	q.lock.Lock()
	defer q.lock.Unlock()
	now := q.clock.Now()
	var items []types.UID
	for k, v := range q.queue {
        // 遍历队列如果过期时间<当前时间，则弹出元素
		if v.Before(now) {
			items = append(items, k)
			delete(q.queue, k)
		}
	}
	return items
}

func (q *basicWorkQueue) Enqueue(item types.UID, delay time.Duration) {
	q.lock.Lock()
	defer q.lock.Unlock()
	q.queue[item] = q.clock.Now().Add(delay)
}
```