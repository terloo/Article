# Manager
Manager是kubelet中提供的大部分Manager的基础Manager，实现类为cacheBasedManager

## 接口
```go
// pkg/kubelet/util/manager/manager.go
type Manager interface {
	// 按namespace和name获取Manager管理的对象
	GetObject(namespace, name string) (runtime.Object, error)

    // 注册指定Pod中所有Manager管理的对象，需要幂等并非阻塞
	RegisterPod(pod *v1.Pod)

	// 取消注册指定Pod中所有Manager管理的对象，需要幂等并非阻塞
	UnregisterPod(pod *v1.Pod)
}

// cacheBasedManager的存储结构接口
type Store interface {
	// AddReference adds a reference to the object to the store.
	// Note that multiple additions to the store has to be allowed
	// in the implementations and effectively treated as refcounted.
	AddReference(namespace, name string)
	// DeleteReference deletes reference to the object from the store.
	// Note that object should be deleted only when there was a
	// corresponding Delete call for each of Add calls (effectively
	// when refcount was reduced to zero).
	DeleteReference(namespace, name string)
	// Get an object from a store.
	Get(namespace, name string) (runtime.Object, error)
}
```