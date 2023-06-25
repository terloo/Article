# container.Cache
Cache接口用于缓存kubelet从CRI中获取到的所有可见的**pods中container的Status**。并提供了一个可以订阅某Pod在某时刻之后状态的方法  
如果没命中缓存，Cache返回一个空的PodStatus对象  
container.Cache会被PLEG定时刷新缓存，在PodWorker中使用此缓存来获取Pod在node上所有container的状态，并根据此状态生成v1.Pod.Status，更新到StatusManager

## 接口
```go
// pkg/kubelet/container/cache.go
type Cache interface {
	Get(types.UID) (*PodStatus, error)
	Set(types.UID, *PodStatus, error, time.Time)
	// GetNewerThan is a blocking call that only returns the status
	// when it is newer than the given time.
	GetNewerThan(types.UID, time.Time) (*PodStatus, error)
	Delete(types.UID)
	UpdateTime(time.Time)
}
```
1. Get：通过UID来获取PodStatus
2. Set：更新UID对应的PodStatus，需要传入一个更新的时间
3. GetNewerThan：返回UUID对应的PodStatus，该PodStatus必须比指定的时间更新。如果还没有，则阻塞该方法
4. Delete：删除UID对应的缓存
5. UpdateTime：修改全局更新时间字段

## 实现类
```go
// pkg/kubelet/container/cache.go
// PodStatus的再包装
type data struct {
	// Status of the pod.
	status *PodStatus
	// Error got when trying to inspect the pod.
	err error
	// Time when the data was last modified.
    // 数据的最近更新时间
	modified time.Time
}

type subRecord struct {
	time time.Time
	ch   chan *data
}

// cache implements Cache.
type cache struct {
	// Lock which guards all internal data structures.
	lock sync.RWMutex
	// Map that stores the pod statuses.
	pods map[types.UID]*data
	// A global timestamp represents how fresh the cached data is. All
	// cache content is at the least newer than this timestamp. Note that the
	// timestamp is nil after initialization, and will only become non-nil when
	// it is ready to serve the cached statuses.
	timestamp *time.Time
	// Map that stores the subscriber records.
	subscribers map[types.UID][]*subRecord
}
```
1. timestamp：用于表示缓存新鲜程度的时间戳，所有的缓存数据都应该比此时间戳新。在初始化时为nil，在准备好之后不为nil
2. subscribers：保存订阅者信息，在PodStatus比subRecord.time更新后，向subRecord.ch中传入data

## 构建函数
```go
// pkg/kubelet/container/cache.go
func NewCache() Cache {
	return &cache{pods: map[types.UID]*data{}, subscribers: map[types.UID][]*subRecord{}}
}
```

## 增删改查
```go
// pkg/kubelet/container/cache.go
func (c *cache) Get(id types.UID) (*PodStatus, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()
	d := c.get(id)
	return d.status, d.err
}

func (c *cache) get(id types.UID) *data {
	d, ok := c.pods[id]
	if !ok {
		// Cache should store *all* pod/container information known by the
		// container runtime. A cache miss indicates that there are no states
		// regarding the pod last time we queried the container runtime.
		// What this *really* means is that there are no visible pod/containers
		// associated with this pod. Simply return an default (mostly empty)
		// PodStatus to reflect this.
		return makeDefaultData(id)
	}
	return d
}

// 没有命中缓存的话，则返回ContainerStatuses为nil的PodStatus
func makeDefaultData(id types.UID) *data {
	return &data{status: &PodStatus{ID: id}, err: nil}
}

func (c *cache) Delete(id types.UID) {
	c.lock.Lock()
	defer c.lock.Unlock()
	delete(c.pods, id)
}

func (c *cache) Set(id types.UID, status *PodStatus, err error, timestamp time.Time) {
	c.lock.Lock()
	defer c.lock.Unlock()
    // 通知所有的订阅者
	defer c.notify(id, timestamp)
	c.pods[id] = &data{status: status, err: err, modified: timestamp}
}

func (c *cache) Get(id types.UID) (*PodStatus, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()
	d := c.get(id)
	return d.status, d.err
}
```

## GetNewerThan
订阅一个UUID对应的PodStatus，只有在PodStatus的更新时间大于传入的时间时返回该PodStatus，否则阻塞
```go
// pkg/kubelet/container/cache.go
func (c *cache) GetNewerThan(id types.UID, minTime time.Time) (*PodStatus, error) {
	ch := c.subscribe(id, minTime)
    // 阻塞获取ch中的数据
	d := <-ch
	return d.status, d.err
}

func (c *cache) subscribe(id types.UID, timestamp time.Time) chan *data {
	ch := make(chan *data, 1)
	c.lock.Lock()
	defer c.lock.Unlock()
	d := c.getIfNewerThan(id, timestamp)
    // 如果能直接得到PodStatus，则直接传入ch，返回ch
	if d != nil {
		// If the cache entry is ready, send the data and return immediately.
		ch <- d
		return ch
	}
	// Add the subscription record.
    // 否则的话，将ch加入订阅者缓存中，返回ch
	c.subscribers[id] = append(c.subscribers[id], &subRecord{time: timestamp, ch: ch})
	return ch
}

// 如果PodStatus的更新时间大于传入的时间，则返回PodStatus，否则返回nil
func (c *cache) getIfNewerThan(id types.UID, minTime time.Time) *data {
	d, ok := c.pods[id]
	globalTimestampIsNewer := (c.timestamp != nil && c.timestamp.After(minTime))
	if !ok && globalTimestampIsNewer {
		// Status is not cached, but the global timestamp is newer than
		// minTime, return the default status.
		return makeDefaultData(id)
	}
	if ok && (d.modified.After(minTime) || globalTimestampIsNewer) {
		// Status is cached, return status if either of the following is true.
		//   * status was modified after minTime
		//   * the global timestamp of the cache is newer than minTime.
		return d
	}
	// The pod status is not ready.
	return nil
}
```

## notify
在进行Set操作后，需要通知所有的订阅者，判断能否返回订阅者的GetNewerThan方法
```go
// pkg/kubelet/container/cache.go
func (c *cache) notify(id types.UID, timestamp time.Time) {
	list, ok := c.subscribers[id]
	if !ok {
		// No one to notify.
		return
	}
    // newList保存了还无法进行通知的订阅者列表
	newList := []*subRecord{}
	for i, r := range list {
		if timestamp.Before(r.time) {
			// Doesn't meet the time requirement; keep the record.
			newList = append(newList, list[i])
			continue
		}
		r.ch <- c.get(id)
		close(r.ch)
	}
	if len(newList) == 0 {
		delete(c.subscribers, id)
	} else {
		c.subscribers[id] = newList
	}
}
```