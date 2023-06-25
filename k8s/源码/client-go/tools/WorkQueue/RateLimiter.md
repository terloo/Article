# RateLimiter
限速器，是实现限速队列的核心组件

## 接口
```go
// vendor/k8s.io/client-go/util/workqueue/default_limiter.go
type RateLimiter interface {
	// When gets an item and gets to decide how long that item should wait
	When(item interface{}) time.Duration
	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for failing
	// or for success, we'll stop tracking it
	Forget(item interface{})
	// NumRequeues returns back how many failures the item has had
	NumRequeues(item interface{}) int
}
```
1. When：获取该元素在被添加到队列时应等待多长时间
2. Forget：停止元素重复入队次数记录
3. NumRequeues：返回该元素的重复入队次数

## 实现类
1. MaxOfRateLimiter：限速器组合，将会使用其内部所有限速器中最大的限速值
2. BucketRateLimiter：令牌桶限速器，基础限速器，限制所有元素在短时间内的添加速度
3. ItemExponentialFailureRateLimiter：指数失败限速器，限制同一个元素在短时间内的添加速度
```go
// vendor/k8s.io/client-go/util/workqueue/default_limiter.go
type MaxOfRateLimiter struct {
	limiters []RateLimiter
}

// 底层由golang.org/x/time/rate提供实现，此结构体是适配器
type BucketRateLimiter struct {
	*rate.Limiter
}

// 同一
type ItemExponentialFailureRateLimiter struct {
	failuresLock sync.Mutex
	failures     map[interface{}]int

	baseDelay time.Duration
	maxDelay  time.Duration
}
```

## 构造函数
```go
// vendor/k8s.io/client-go/util/workqueue/default_limiter.go
// 默认的限速器，是令牌桶限速器和指数失败限速器的组合
func DefaultControllerRateLimiter() RateLimiter {
	return NewMaxOfRateLimiter(
		NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
		// 10 qps, 100 bucket size.  This is only for retry speed and its only the overall factor (not per item)
		&BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
	)
}
```