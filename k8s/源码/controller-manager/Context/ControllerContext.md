# ControllerContext
ControllerContext定义了所有Controller启动时需要使用到的上下文对象

## 结构体
```go
// cmd/kube-controller-manager/app/controllermanager.go
type ControllerContext struct {
	// ClientBuilder will provide a client for this controller to use
	ClientBuilder clientbuilder.ControllerClientBuilder

	// InformerFactory gives access to informers for the controller.
	InformerFactory informers.SharedInformerFactory

	// ObjectOrMetadataInformerFactory gives access to informers for typed resources
	// and dynamic resources by their metadata. All generic controllers currently use
	// object metadata - if a future controller needs access to the full object this
	// would become GenericInformerFactory and take a dynamic client.
	ObjectOrMetadataInformerFactory informerfactory.InformerFactory

	// ComponentConfig provides access to init options for a given controller
	ComponentConfig kubectrlmgrconfig.KubeControllerManagerConfiguration

	// DeferredDiscoveryRESTMapper is a RESTMapper that will defer
	// initialization of the RESTMapper until the first mapping is
	// requested.
	RESTMapper *restmapper.DeferredDiscoveryRESTMapper

	// AvailableResources is a map listing currently available resources
	AvailableResources map[schema.GroupVersionResource]bool

	// Cloud is the cloud provider interface for the controllers to use.
	// It must be initialized and ready to use.
	Cloud cloudprovider.Interface

	// Control for which control loops to be run
	// IncludeCloudLoops is for a kube-controller-manager running all loops
	// ExternalLoops is for a kube-controller-manager running with a cloud-controller-manager
	LoopMode ControllerLoopMode

	// InformersStarted is closed after all of the controllers have been initialized and are running.  After this point it is safe,
	// for an individual controller to start the shared informers. Before it is closed, they should not.
	InformersStarted chan struct{}

	// ResyncPeriod generates a duration each time it is invoked; this is so that
	// multiple controllers don't get into lock-step and all hammer the apiserver
	// with list requests simultaneously.
	ResyncPeriod func() time.Duration
}
```

## 构建函数
传入ControllerClientBuilder，构造ControllerContext
1. rootClientBuilder：通用的clinetBuilder
2. clientBuilder：各个Controller独自使用的clientBuilder，在不指定`use-service-account-credentials`选项时，和rootClientBuilder是一个对象
```go
// cmd/kube-controller-manager/app/controllermanager.go
func CreateControllerContext(s *config.CompletedConfig, rootClientBuilder, clientBuilder clientbuilder.ControllerClientBuilder, stop <-chan struct{}) (ControllerContext, error) {
	// 由rootClientBuilder构建出的用于构建informers的clientset
	versionedClient := rootClientBuilder.ClientOrDie("shared-informers")
	// 构建sharedInformersFactory
	sharedInformers := informers.NewSharedInformerFactory(versionedClient, ResyncPeriod(s)())

	// 由rootClientBuilder构建出的用于构建metadata-informers的clientset
	metadataClient := metadata.NewForConfigOrDie(rootClientBuilder.ConfigOrDie("metadata-informers"))
	metadataInformers := metadatainformer.NewSharedInformerFactory(metadataClient, ResyncPeriod(s)())

	// If apiserver is not running we should wait for some time and fail only then. This is particularly
	// important when we start apiserver and controller manager at the same time.
	// 使用versionedClient来测试api-server的/healz接口
	if err := genericcontrollermanager.WaitForAPIServer(versionedClient, 10*time.Second); err != nil {
		return ControllerContext{}, fmt.Errorf("failed to wait for apiserver being healthy: %v", err)
	}

	// Use a discovery client capable of being refreshed.
	// 构建MemCacheClient用于服务发现
	discoveryClient := rootClientBuilder.DiscoveryClientOrDie("controller-discovery")
	cachedClient := cacheddiscovery.NewMemCacheClient(discoveryClient)
	// 构建restMapper
	restMapper := restmapper.NewDeferredDiscoveryRESTMapper(cachedClient)
	// 每30s刷新一次restMapper缓存
	go wait.Until(func() {
		restMapper.Reset()
	}, 30*time.Second, stop)

	availableResources, err := GetAvailableResources(rootClientBuilder)
	if err != nil {
		return ControllerContext{}, err
	}

	cloud, loopMode, err := createCloudProvider(s.ComponentConfig.KubeCloudShared.CloudProvider.Name, s.ComponentConfig.KubeCloudShared.ExternalCloudVolumePlugin,
		s.ComponentConfig.KubeCloudShared.CloudProvider.CloudConfigFile, s.ComponentConfig.KubeCloudShared.AllowUntaggedCloud, sharedInformers)
	if err != nil {
		return ControllerContext{}, err
	}

	ctx := ControllerContext{
		ClientBuilder:                   clientBuilder,
		InformerFactory:                 sharedInformers,
		ObjectOrMetadataInformerFactory: informerfactory.NewInformerFactory(sharedInformers, metadataInformers),
		ComponentConfig:                 s.ComponentConfig,
		RESTMapper:                      restMapper,
		AvailableResources:              availableResources,
		Cloud:                           cloud,
		LoopMode:                        loopMode,
		InformersStarted:                make(chan struct{}),
		ResyncPeriod:                    ResyncPeriod(s),
	}
	return ctx, nil
}
```