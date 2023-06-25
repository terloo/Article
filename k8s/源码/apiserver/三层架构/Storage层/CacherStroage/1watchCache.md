# watchCache
CacherStorage在实例化过程中会使用Reflector框架的ListAndWatch函数通过UnderlyingStorage监控Etcd集群的Watch事件  

## 处理事件
watchCache接受Reflector框架的事件回调，并实现了Add、Update、Delete方法，分别用于接收watch.Added、watch.Modified、watch.Deleted事件。通过goroutine(即w.processEvent)将事件存储到以下三个地方：
1. w.onEvent：将事件回调给CacherStorage，CacherStorage将其分发给目前所有已连接的观察者，该过程通过非阻塞机制实现
2. w.cache：将事件回调给缓存滑动窗口，它提供了对Watch操作的缓存数据，防止因为网络等原因导致的客户端事件丢失
3. cache.Store：将事件存储至本地缓存，cache.Store与client-go下的Indexer功能相同。

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go
type watchCache struct {
	sync.RWMutex

	// Condition on which lists are waiting for the fresh enough
	// resource version.
	cond *sync.Cond

	// Maximum size of history window.
	capacity int

	// upper bound of capacity since event cache has a dynamic size.
	upperBoundCapacity int

	// lower bound of capacity since event cache has a dynamic size.
	lowerBoundCapacity int

	// keyFunc is used to get a key in the underlying storage for a given object.
	keyFunc func(runtime.Object) (string, error)

	// getAttrsFunc is used to get labels and fields of an object.
	getAttrsFunc func(runtime.Object) (labels.Set, fields.Set, error)

	// cache is used a cyclic buffer - its first element (with the smallest
	// resourceVersion) is defined by startIndex, its last element is defined
	// by endIndex (if cache is full it will be startIndex + capacity).
	// Both startIndex and endIndex can be greater than buffer capacity -
	// you should always apply modulo capacity to get an index in cache array.
	cache      []*watchCacheEvent
	startIndex int
	endIndex   int

	// store will effectively support LIST operation from the "end of cache
	// history" i.e. from the moment just after the newest cached watched event.
	// It is necessary to effectively allow clients to start watching at now.
	// NOTE: We assume that <store> is thread-safe.
    // client-go的Indexer框架
	store cache.Indexer

	// ResourceVersion up to which the watchCache is propagated.
	resourceVersion uint64

	// ResourceVersion of the last list result (populated via Replace() method).
	listResourceVersion uint64

	// This handler is run at the end of every successful Replace() method.
	onReplace func()

	// This handler is run at the end of every Add/Update/Delete method
	// and additionally gets the previous value of the object.
	eventHandler func(*watchCacheEvent)

	// for testing timeouts.
	clock clock.Clock

	// An underlying storage.Versioner.
	versioner storage.Versioner

	// cacher's objectType.
	objectType reflect.Type
}
```

## processEvent
将
```go
// vendor/k8s.io/apiserver/pkg/storage/cacher/watch_cache.go
func (w *watchCache) processEvent(event watch.Event, resourceVersion uint64, updateFunc func(*storeElement) error) error {
	key, err := w.keyFunc(event.Object)
	if err != nil {
		return fmt.Errorf("couldn't compute key: %v", err)
	}
	elem := &storeElement{Key: key, Object: event.Object}
	elem.Labels, elem.Fields, err = w.getAttrsFunc(event.Object)
	if err != nil {
		return err
	}

	wcEvent := &watchCacheEvent{
		Type:            event.Type,
		Object:          elem.Object,
		ObjLabels:       elem.Labels,
		ObjFields:       elem.Fields,
		Key:             key,
		ResourceVersion: resourceVersion,
		RecordTime:      w.clock.Now(),
	}

	if err := func() error {
		// TODO: We should consider moving this lock below after the watchCacheEvent
		// is created. In such situation, the only problematic scenario is Replace(
		// happening after getting object from store and before acquiring a lock.
		// Maybe introduce another lock for this purpose.
		w.Lock()
		defer w.Unlock()

		previous, exists, err := w.store.Get(elem)
		if err != nil {
			return err
		}
		if exists {
			previousElem := previous.(*storeElement)
			wcEvent.PrevObject = previousElem.Object
			wcEvent.PrevObjLabels = previousElem.Labels
			wcEvent.PrevObjFields = previousElem.Fields
		}


        // 1.发送到滑动窗口
		w.updateCache(wcEvent)
        // 更新resourceVersion
		w.resourceVersion = resourceVersion
        // 通知消费者消费
		defer w.cond.Broadcast()

        // 2.发送到本地缓存(Indexer)
		return updateFunc(elem)
	}(); err != nil {
		return err
	}

	// Avoid calling event handler under lock.
	// This is safe as long as there is at most one call to Add/Update/Delete and
	// UpdateResourceVersion in flight at any point in time, which is true now,
	// because reflector calls them synchronously from its main thread.
	if w.eventHandler != nil {
		w.eventHandler(wcEvent)
	}
	return nil
}
```