# watchBasedManager
watchBasedManager并无实际结构体，是Store实现类为objectStore的cacheBasedManager

## 构造函数
```go
// pkg/kubelet/util/manager/watch_based_manager.go
func NewWatchBasedManager(
	listObject listObjectFunc,
	watchObject watchObjectFunc,
	newObject newObjectFunc,
	isImmutable isImmutableFunc,
	groupResource schema.GroupResource,
	resyncInterval time.Duration,
	getReferencedObjects func(*v1.Pod) sets.String) Manager {

	// If a configmap/secret is used as a volume, the volumeManager will visit the objectCacheItem every resyncInterval cycle,
	// We just want to stop the objectCacheItem referenced by environment variables,
	// So, maxIdleTime is set to an integer multiple of resyncInterval,
	// We currently set it to 5 times.
    // objectStore的reflector最大空间时间为同步周期的5倍
	maxIdleTime := resyncInterval * 5

	objectStore := NewObjectCache(listObject, watchObject, newObject, isImmutable, groupResource, clock.RealClock{}, maxIdleTime)
	return NewCacheBasedManager(objectStore, getReferencedObjects)
}
```