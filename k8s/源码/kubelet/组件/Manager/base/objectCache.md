# objectCache
objectCache是Store的实现类，在watchBasedManager中用于watch机制的本地缓存  
其中维护了`资源对象 -> 资源对象的Refletor`的映射

## 结构体
```go
// pkg/kubelet/util/manager/watch_based_manager.go
type objectCache struct {
	listObject    listObjectFunc
	watchObject   watchObjectFunc
	newObject     newObjectFunc
	isImmutable   isImmutableFunc
	groupResource schema.GroupResource
	clock         clock.Clock
	maxIdleTime   time.Duration

	lock  sync.RWMutex
	items map[objectKey]*objectCacheItem
}
```
1. listObject、watchObject：基于ClientSet的list和watch函数
2. newObject：新建资源对象的函数
3. groupResource：缓存的资源对象GR
4. lock：保护items并发安全的锁
5. items：`资源对象 -> 资源对象的Refletor`的映射

## 构造函数
```go
// pkg/kubelet/util/manager/watch_based_manager.go
const minIdleTime = 1 * time.Minute

// NewObjectCache returns a new watch-based instance of Store interface.
func NewObjectCache(
	listObject listObjectFunc,
	watchObject watchObjectFunc,
	newObject newObjectFunc,
	isImmutable isImmutableFunc,
	groupResource schema.GroupResource,
	clock clock.Clock,
	maxIdleTime time.Duration) Store {

	if maxIdleTime < minIdleTime {
		maxIdleTime = minIdleTime
	}

	store := &objectCache{
		listObject:    listObject,
		watchObject:   watchObject,
		newObject:     newObject,
		isImmutable:   isImmutable,
		groupResource: groupResource,
		clock:         clock,
		maxIdleTime:   maxIdleTime,
		items:         make(map[objectKey]*objectCacheItem),
	}

	// TODO propagate stopCh from the higher level.
	go wait.Until(store.startRecycleIdleWatch, time.Minute, wait.NeverStop)
	return store
}

// 定时检查是否有资源对象的reflector空闲时间过长，如果过长，将其关闭
func (c *objectCache) startRecycleIdleWatch() {
	c.lock.Lock()
	defer c.lock.Unlock()

	for key, item := range c.items {
		if item.stopIfIdle(c.clock.Now(), c.maxIdleTime) {
			klog.V(4).InfoS("Not acquired for long time, Stopped watching for changes", "objectKey", key, "maxIdleTime", c.maxIdleTime)
		}
	}
}
```

## AddReference
传入资源对象的namespace和name，创建reflector进行管理。由于此方法会被RegisterPod所调用，所以尽量不进行阻塞操作
```go
// pkg/kubelet/util/manager/watch_based_manager.go
func (c *objectCache) AddReference(namespace, name string) {
	key := objectKey{namespace: namespace, name: name}

	// AddReference is called from RegisterPod thus it needs to be efficient.
	// Thus, it is only increasing refCount and in case of first registration
	// of a given object it starts corresponding reflector.
	// It's responsibility of the first Get operation to wait until the
	// reflector propagated the store.
	c.lock.Lock()
	defer c.lock.Unlock()
	item, exists := c.items[key]
	if !exists {
		// 为每个资源对象都创建一个reflector
		item = c.newReflector(namespace, name)
		c.items[key] = item
	}
	item.refCount++
}

// newReflector实质上是objectCacheItem的构造函数
func (c *objectCache) newReflector(namespace, name string) *objectCacheItem {
	// list和watch时使用name进行筛选
	fieldSelector := fields.Set{"metadata.name": name}.AsSelector().String()
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		options.FieldSelector = fieldSelector
		return c.listObject(namespace, options)
	}
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.FieldSelector = fieldSelector
		return c.watchObject(namespace, options)
	}
	// 构建cacheStore
	store := c.newStore()
	reflector := cache.NewNamedReflector(
		fmt.Sprintf("object-%q/%q", namespace, name),
		&cache.ListWatch{ListFunc: listFunc, WatchFunc: watchFunc},
		c.newObject(),
		store,
		0,
	)
	item := &objectCacheItem{
		refCount:  0,
		store:     store,
		reflector: reflector,
		hasSynced: func() (bool, error) { return store.hasSynced(), nil },
		stopCh:    make(chan struct{}),
	}
	// 启动item的Reflector
	go item.startReflector()
	return item
}

// 先构建一个Indexer，再构建一个cacheStore
func (c *objectCache) newStore() *cacheStore {
	// TODO: We may consider created a dedicated store keeping just a single
	// item, instead of using a generic store implementation for this purpose.
	// However, simple benchmarks show that memory overhead in that case is
	// decrease from ~600B to ~300B per object. So we are not optimizing it
	// until we will see a good reason for that.
	store := cache.NewStore(cache.MetaNamespaceKeyFunc)
	return &cacheStore{store, sync.Mutex{}, false}
}
```

## DeleteReference
```go
// pkg/kubelet/util/manager/watch_based_manager.go
func (c *objectCache) DeleteReference(namespace, name string) {
	key := objectKey{namespace: namespace, name: name}

	c.lock.Lock()
	defer c.lock.Unlock()
	if item, ok := c.items[key]; ok {
		item.refCount--
		if item.refCount == 0 {
			// 如果引用计数为0，则停止底层reflector
			// Stop the underlying reflector.
			item.stop()
			delete(c.items, key)
		}
	}
}
```

## Get
```go
// pkg/kubelet/manager/watch_based_manager.go
func (c *objectCache) Get(namespace, name string) (runtime.Object, error) {
	key := objectKey{namespace: namespace, name: name}

	c.lock.RLock()
	item, exists := c.items[key]
	c.lock.RUnlock()

	if !exists {
		return nil, fmt.Errorf("object %q/%q not registered", namespace, name)
	}
	// Record last access time independently if it succeeded or not.
	// This protects from premature (racy) reflector closure.
	// 首先更新最新访问时间
	item.setLastAccessTime(c.clock.Now())

	// 如果reflector已经关闭，则先重启
	item.restartReflectorIfNeeded()
	// 等待重启完成
	if err := wait.PollImmediate(10*time.Millisecond, time.Second, item.hasSynced); err != nil {
		return nil, fmt.Errorf("failed to sync %s cache: %v", c.groupResource.String(), err)
	}
	// 使用namespace/name的方式从本地缓存(cacheStore，即Indexer的扩展类)中获取对象
	obj, exists, err := item.store.GetByKey(c.key(namespace, name))
	if err != nil {
		return nil, err
	}
	if !exists {
		return nil, apierrors.NewNotFound(c.groupResource, name)
	}
	if object, ok := obj.(runtime.Object); ok {
		// If the returned object is immutable, stop the reflector.
		// NOTE: we may potentially not even start the reflector if the object is
		// already immutable. However, given that:
		// - we want to also handle the case when object is marked as immutable later
		// - Secrets and ConfigMaps are periodically fetched by volumemanager anyway
		// - doing that wouldn't provide visible scalability/performance gain - we
		//   already have it from here
		// - doing that would require significant refactoring to reflector
		// we limit ourselves to just quickly stop the reflector here.
		// 调用isImmutable函数来判断资源对象是否可变，如果不可变，则直接停止资源对象对应的reflector
		if c.isImmutable(object) {
			item.setImmutable()
			if item.stop() {
				klog.V(4).InfoS("Stopped watching for changes - object is immutable", "obj", klog.KRef(namespace, name))
			}
		}
		return object, nil
	}
	return nil, fmt.Errorf("unexpected object type: %v", obj)
}
```