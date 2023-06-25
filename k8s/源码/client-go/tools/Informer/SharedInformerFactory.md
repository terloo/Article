# SharedInformerFactory
SharedInformerFactory用于获取所有内置资源和CRD的Informer，由于其内部有缓存，多次获取同种资源的Informer时将会直接返回已创建的Informer

## 接口
```go
// vendor/k8s.io/client-go/informers/factory.go
type SharedInformerFactory interface {
	internalinterfaces.SharedInformerFactory
    // 获取传入的GVR的Informer，GVR必须是k8s内置资源
	ForResource(resource schema.GroupVersionResource) (GenericInformer, error)
    // 等待所有Informer的缓存同步完毕
	WaitForCacheSync(stopCh <-chan struct{}) map[reflect.Type]bool

    // 获取指定Group的Interface，通过该Interface获取指定Version的Informer
	Admissionregistration() admissionregistration.Interface
	Internal() apiserverinternal.Interface
	Apps() apps.Interface
	Autoscaling() autoscaling.Interface
	Batch() batch.Interface
	Certificates() certificates.Interface
	Coordination() coordination.Interface
	Core() core.Interface
	Discovery() discovery.Interface
	Events() events.Interface
	Extensions() extensions.Interface
	Flowcontrol() flowcontrol.Interface
	Networking() networking.Interface
	Node() node.Interface
	Policy() policy.Interface
	Rbac() rbac.Interface
	Scheduling() scheduling.Interface
	Storage() storage.Interface
}

// 定义InformerFacotry的基本操作
type SharedInformerFactory interface {
    // 启动通过这个Factory创建的所有Informer，这个方法是非阻塞的
	Start(stopCh <-chan struct{})
    // 通过NewInformerFunc来构建一个informer
	InformerFor(obj runtime.Object, newFunc NewInformerFunc) cache.SharedIndexInformer
}

// kubernetes.Interface实例一般是clientSet
type NewInformerFunc func(kubernetes.Interface, time.Duration) cache.SharedIndexInformer
```

## 实现类
```go
// vendor/k8s.io/client-go/informers/factory.go
type sharedInformerFactory struct {
    // ClientSet
	client           kubernetes.Interface
    // 指定命名空间，空字符串为所有命名空间。通过WithNamespace进行指定
	namespace        string
    // 修改ListOptions的函数
	tweakListOptions internalinterfaces.TweakListOptionsFunc
	lock             sync.Mutex
    // 默认重同步周期
	defaultResync    time.Duration
    // 自定义重同步周期
	customResync     map[reflect.Type]time.Duration

    // informers缓存
	informers map[reflect.Type]cache.SharedIndexInformer
	// startedInformers is used for tracking which informers have been started.
	// This allows Start() to be called multiple times safely.
    // 保存Informers是否启动
	startedInformers map[reflect.Type]bool
}
```

## 构造函数
```go
// vendor/k8s.io/client-go/informers/factory.go
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
	factory := &sharedInformerFactory{
		client:           client,
        // 默认所有命名空间
		namespace:        v1.NamespaceAll,
		defaultResync:    defaultResync,
		informers:        make(map[reflect.Type]cache.SharedIndexInformer),
		startedInformers: make(map[reflect.Type]bool),
		customResync:     make(map[reflect.Type]time.Duration),
	}

    // 应用所有的options，比如指定命名空间，自定义重同步周期
	// Apply all options
	for _, opt := range options {
		factory = opt(factory)
	}

	return factory
}
```

## InformerFor
这个方法一般由各个GVR中编写的获取Informer的方法调用，获取CRD的Informer也通过这个方法获取
```go
// vendor/k8s.io/client-go/infromers/factory.go
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	informerType := reflect.TypeOf(obj)
    // 如果Informer已创建，则直接从缓存获取
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

    // 获取自定义同步周期
	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}

    // 调用newFunc来获取Informer
	informer = newFunc(f.client, resyncPeriod)
    // 缓存Informer
	f.informers[informerType] = informer

	return informer
}
``` 