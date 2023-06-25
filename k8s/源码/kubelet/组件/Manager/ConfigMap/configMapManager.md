# configMapManager
configMapManager缓存了所有已注册的Pod的所有Config，并使用Reflect机制进行实时更新，pod更新后，kubelet可以从此Manager获取最新的Config配置。  
底层由WatchBasedManager进行实现

## 接口
```go
// pkg/kubelet/configmap/configmap_manager.go
type Manager interface {
	// Get configmap by configmap namespace and name.
	GetConfigMap(namespace, name string) (*v1.ConfigMap, error)

	// WARNING: Register/UnregisterPod functions should be efficient,
	// i.e. should not block on network operations.

	// RegisterPod registers all configmaps from a given pod.
	RegisterPod(pod *v1.Pod)

	// UnregisterPod unregisters configmaps from a given pod that are not
	// used by any other registered pod.
	UnregisterPod(pod *v1.Pod)
}
```

## 实现类
```go
// pkg/kubelet/configmap/configmap_manager.go
type configMapManager struct {
	manager manager.Manager
}
```

## 构造函数
```go
// pkg/kubelet/configmap/configmap_manager.go
func NewWatchingConfigMapManager(kubeClient clientset.Interface, resyncInterval time.Duration) Manager {
	listConfigMap := func(namespace string, opts metav1.ListOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).List(context.TODO(), opts)
	}
	watchConfigMap := func(namespace string, opts metav1.ListOptions) (watch.Interface, error) {
		return kubeClient.CoreV1().ConfigMaps(namespace).Watch(context.TODO(), opts)
	}
	newConfigMap := func() runtime.Object {
		return &v1.ConfigMap{}
	}
	isImmutable := func(object runtime.Object) bool {
		if configMap, ok := object.(*v1.ConfigMap); ok {
			return configMap.Immutable != nil && *configMap.Immutable
		}
		return false
	}
	gr := corev1.Resource("configmap")
	return &configMapManager{
		manager: manager.NewWatchBasedManager(listConfigMap, watchConfigMap, newConfigMap, isImmutable, gr, resyncInterval, getConfigMapNames),
	}
}

func getConfigMapNames(pod *v1.Pod) sets.String {
	result := sets.NewString()
	podutil.VisitPodConfigmapNames(pod, func(name string) bool {
		result.Insert(name)
		return true
	})
	return result
}
```

## 方法
```go
// pkg/kubelet/config/configmap_manager.go

// 从watchBasedManager获取对象后进行接口断言
func (c *configMapManager) GetConfigMap(namespace, name string) (*v1.ConfigMap, error) {
	object, err := c.manager.GetObject(namespace, name)
	if err != nil {
		return nil, err
	}
	if configmap, ok := object.(*v1.ConfigMap); ok {
		return configmap, nil
	}
	return nil, fmt.Errorf("unexpected object type: %v", object)
}

// 注册与取消注册直接调用底层watchBasedManager进行实现
func (c *configMapManager) RegisterPod(pod *v1.Pod) {
	c.manager.RegisterPod(pod)
}

func (c *configMapManager) UnregisterPod(pod *v1.Pod) {
	c.manager.UnregisterPod(pod)
}
```