# RateLimitingQueue
限速队列可以在向队列添加元素时限制添加速度。添加元素时通过限速器计算出该元素的延迟时间，再通过延迟队列进行延迟添加

## 接口
```go
// vendor/k8s.io/client-go/util/workqueue/rate_limiting_queue.go
type RateLimitingInterface interface {
	// 延迟队列
	DelayingInterface

	// AddRateLimited adds an item to the workqueue after the rate limiter says it's ok
	// 向队列中添加一个元素，限速
	AddRateLimited(item interface{})

	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for perm failing
	// or for success, we'll stop the rate limiter from tracking it.  This only clears the `rateLimiter`, you
	// still have to call `Done` on the queue.
	// 停止元素重复入队次数记录
	Forget(item interface{})

	// NumRequeues returns back how many times the item was requeued
	// 获取该元素被重复入队了多少次
	NumRequeues(item interface{}) int
}
```

## 实现类
```go
// vendor/k8s.io/client-go/util/workqueue/rate_limiting_queue.go
type rateLimitingType struct {
	DelayingInterface

	// 限速器
	rateLimiter RateLimiter
}
```

## 构造函数
```go
// vendor/k8s.io/client-go/util/workqueue/rate_limiting_queue.go
func NewRateLimitingQueue(rateLimiter RateLimiter) RateLimitingInterface {
	return &rateLimitingType{
		// 直接构造一个延迟队列
		DelayingInterface: NewDelayingQueue(),
		rateLimiter:       rateLimiter,
	}
}
```

## 方法
```go
// vendor/k8s.io/client-go/util/workqueue/rate_limiting_queue.go
func (q *rateLimitingType) AddRateLimited(item interface{}) {
	// 通过限速器计算出延迟时间，再添加到延迟队列
	q.DelayingInterface.AddAfter(item, q.rateLimiter.When(item))
}

func (q *rateLimitingType) NumRequeues(item interface{}) int {
	return q.rateLimiter.NumRequeues(item)
}

func (q *rateLimitingType) Forget(item interface{}) {
	q.rateLimiter.Forget(item)
}
```
