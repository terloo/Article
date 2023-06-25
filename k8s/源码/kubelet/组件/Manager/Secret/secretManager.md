# secretManager
secretManager缓存了所有已注册的Pod的所有Secret，并使用Reflect机制进行实时更新，pod更新后，kubelet可以从此Manager获取最新的Secret配置。  
底层由WatchBasedManager进行实现

## 接口
```go
// pkg/kublet/secret/secret_manager.go
type Manager interface {
	// Get secret by secret namespace and name.
	GetSecret(namespace, name string) (*v1.Secret, error)

	// WARNING: Register/UnregisterPod functions should be efficient,
	// i.e. should not block on network operations.

	// RegisterPod registers all secrets from a given pod.
	RegisterPod(pod *v1.Pod)

	// UnregisterPod unregisters secrets from a given pod that are not
	// used by any other registered pod.
	UnregisterPod(pod *v1.Pod)
}
```
1. GetSecret：获取指定的Secret
2. RegisterPod：注册指定的Pod中所有的Secret
3. UnregisterPod：取消注册指定的Pod中的所有的Secret

## 实现类
```go
// pkg/kubelet/secret/secret_manager.go
type secretManager struct {
	manager manager.Manager
}
```

## 构造函数
```go
// pkg/kubelet/secret/secret_manager.go
func NewWatchingSecretManager(kubeClient clientset.Interface, resyncInterval time.Duration) Manager {
	listSecret := func(namespace string, opts metav1.ListOptions) (runtime.Object, error) {
		return kubeClient.CoreV1().Secrets(namespace).List(context.TODO(), opts)
	}
	watchSecret := func(namespace string, opts metav1.ListOptions) (watch.Interface, error) {
		return kubeClient.CoreV1().Secrets(namespace).Watch(context.TODO(), opts)
	}
	newSecret := func() runtime.Object {
		return &v1.Secret{}
	}
	// 根据Secret配置来判断是否可变
	isImmutable := func(object runtime.Object) bool {
		if secret, ok := object.(*v1.Secret); ok {
			return secret.Immutable != nil && *secret.Immutable
		}
		return false
	}
	gr := corev1.Resource("secret")
	return &secretManager{
		// getSecretNames从Pod中获取Secret配置
		manager: manager.NewWatchBasedManager(listSecret, watchSecret, newSecret, isImmutable, gr, resyncInterval, getSecretNames),
	}
}

func getSecretNames(pod *v1.Pod) sets.String {
	result := sets.NewString()
	podutil.VisitPodSecretNames(pod, func(name string) bool {
		result.Insert(name)
		return true
	})
	return result
}
```

## 方法
```go
// pkg/kubelet/secret/secret_manager.go
func (s *secretManager) GetSecret(namespace, name string) (*v1.Secret, error) {
	object, err := s.manager.GetObject(namespace, name)
	if err != nil {
		return nil, err
	}
	if secret, ok := object.(*v1.Secret); ok {
		return secret, nil
	}
	return nil, fmt.Errorf("unexpected object type: %v", object)
}

func (s *secretManager) RegisterPod(pod *v1.Pod) {
	s.manager.RegisterPod(pod)
}

func (s *secretManager) UnregisterPod(pod *v1.Pod) {
	s.manager.UnregisterPod(pod)
}
```