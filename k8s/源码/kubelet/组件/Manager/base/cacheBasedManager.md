# cacheBasedManager
cacheBasedManager会缓存所有已注册的Pod，并在Pod注册时，调用Store.AddReference来添加所有该Pod中需要管理的资源对象

## 结构体
```go
// pkg/kubelet/util/manager/cache_based_manager.go
type cacheBasedManager struct {
	objectStore          Store
	getReferencedObjects func(*v1.Pod) sets.String

	lock           sync.Mutex
	registeredPods map[objectKey]*v1.Pod
}

type objectKey struct {
    // Pod的namespace
	namespace string
    // Pod的name
	name      string
}
```
1. objectStore：Store实现类，用于存储缓存。在watchBasedManager中使用objectCache
2. getReferencedObjects：用于从Pod中获取管理对象的名字的函数
3. registeredPods：保存已经注册了的Pod，

## 构造函数
```go
// pkg/kubelet/util/manager/cache_based_manager.go
func NewCacheBasedManager(objectStore Store, getReferencedObjects func(*v1.Pod) sets.String) Manager {
	return &cacheBasedManager{
		objectStore:          objectStore,
		getReferencedObjects: getReferencedObjects,
		registeredPods:       make(map[objectKey]*v1.Pod),
	}
}
```

## GetObject
```go
// pkg/kubelet/util/manager/cache_based_manager.go
func (c *cacheBasedManager) GetObject(namespace, name string) (runtime.Object, error) {
	return c.objectStore.Get(namespace, name)
}
```

## RegisterPod
获取Pod中全部需要管理的对象名字，将其添加到缓存中。如果该Pod已经注册过，需要删除该Pod以前的缓存数据
```go
// pkg/kubelet/util/manager/cache_based_manager.go
func (c *cacheBasedManager) RegisterPod(pod *v1.Pod) {
	names := c.getReferencedObjects(pod)
	c.lock.Lock()
	defer c.lock.Unlock()
	for name := range names {
		c.objectStore.AddReference(pod.Namespace, name)
	}
	var prev *v1.Pod
	key := objectKey{namespace: pod.Namespace, name: pod.Name}
	prev = c.registeredPods[key]
	c.registeredPods[key] = pod
	if prev != nil {
		for name := range c.getReferencedObjects(prev) {
			// On an update, the .Add() call above will have re-incremented the
			// ref count of any existing object, so any objects that are in both
			// names and prev need to have their ref counts decremented. Any that
			// are only in prev need to be completely removed. This unconditional
			// call takes care of both cases.
			c.objectStore.DeleteReference(prev.Namespace, name)
		}
	}
}
```

## UnregisterPod
```go
// pkg/kubelet/util/manager/cache_based_manager.go
func (c *cacheBasedManager) UnregisterPod(pod *v1.Pod) {
	var prev *v1.Pod
	key := objectKey{namespace: pod.Namespace, name: pod.Name}
	c.lock.Lock()
	defer c.lock.Unlock()
	prev = c.registeredPods[key]
	delete(c.registeredPods, key)
	if prev != nil {
		for name := range c.getReferencedObjects(prev) {
			c.objectStore.DeleteReference(prev.Namespace, name)
		}
	}
}
```