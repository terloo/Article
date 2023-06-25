# AggregatorServer

## 所含接口
1. restful接口
   1. /apis/apiregistration.k8s.io/v1/apiservices
   2. /apis/apiregistration.k8s.io/v1/apiservices/status
2. 非restful接口
   1. 全路径：
      1. 除核心组所有G(无论后缀有无`/`)：直接使用apiGroupHandler进行处理
      2. 核心组G(无论后缀有无`/`)：代理到kubeAPIServer进行处理
      3. 所有GV(后缀有无`/`)：代理到kubeAPIServer进行处理
   2. 前缀：
      1. 所有GV(后缀有`/`)：代理到kubeAPIServer进行处理
      2. 所有GVR：代理到kubeAPIServer进行处理

## 结构体
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go
type APIAggregator struct {
	GenericAPIServer *genericapiserver.GenericAPIServer

	// provided for easier embedding
	APIRegistrationInformers informers.SharedInformerFactory

	delegateHandler http.Handler

	// proxyCurrentCertKeyContent holds he client cert used to identify this proxy. Backing APIServices use this to confirm the proxy's identity
	proxyCurrentCertKeyContent certKeyFunc
	proxyTransport             *http.Transport

	// proxyHandlers are the proxy handlers that are currently registered, keyed by apiservice.name
	proxyHandlers map[string]*proxyHandler
	// handledGroups are the groups that already have routes
	handledGroups sets.String

	// lister is used to add group handling for /apis/<group> aggregator lookups based on
	// controller state
	lister listers.APIServiceLister

	// Information needed to determine routing for the aggregator
	serviceResolver ServiceResolver

	// Enable swagger and/or OpenAPI if these configs are non-nil.
	openAPIConfig *openapicommon.Config

	// openAPIAggregationController downloads and merges OpenAPI v2 specs.
	openAPIAggregationController *openapicontroller.AggregationController

	// openAPIV3AggregationController downloads and caches OpenAPI v3 specs.
	openAPIV3AggregationController *openapiv3controller.AggregationController

	// egressSelector selects the proper egress dialer to communicate with the custom apiserver
	// overwrites proxyTransport dialer if not nil
	egressSelector *egressselector.EgressSelector
}
```

## New
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver.go
func (c completedConfig) NewWithDelegate(delegationTarget genericapiserver.DelegationTarget) (*APIAggregator, error) {
    // 构造GenericeServer，名为kube-aggregator
	genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
	if err != nil {
		return nil, err
	}

	apiregistrationClient, err := clientset.NewForConfig(c.GenericConfig.LoopbackClientConfig)
	if err != nil {
		return nil, err
	}
	informerFactory := informers.NewSharedInformerFactory(
		apiregistrationClient,
		5*time.Minute, // this is effectively used as a refresh interval right now.  Might want to do something nicer later on.
	)

	// apiServiceRegistrationControllerInitiated is closed when APIServiceRegistrationController has finished "installing" all known APIServices.
	// At this point we know that the proxy handler knows about APIServices and can handle client requests.
	// Before it might have resulted in a 404 response which could have serious consequences for some controllers like  GC and NS
	//
	// Note that the APIServiceRegistrationController waits for APIServiceInformer to synced before doing its work.
	apiServiceRegistrationControllerInitiated := make(chan struct{})
	if err := genericServer.RegisterMuxAndDiscoveryCompleteSignal("APIServiceRegistrationControllerInitiated", apiServiceRegistrationControllerInitiated); err != nil {
		return nil, err
	}

    // 构造AggregatorServer实例
	s := &APIAggregator{
		GenericAPIServer:           genericServer,
		delegateHandler:            delegationTarget.UnprotectedHandler(),
		proxyTransport:             c.ExtraConfig.ProxyTransport,
		proxyHandlers:              map[string]*proxyHandler{},
		handledGroups:              sets.String{},
		lister:                     informerFactory.Apiregistration().V1().APIServices().Lister(),
		APIRegistrationInformers:   informerFactory,
		serviceResolver:            c.ExtraConfig.ServiceResolver,
		openAPIConfig:              c.GenericConfig.OpenAPIConfig,
		egressSelector:             c.GenericConfig.EgressSelector,
		proxyCurrentCertKeyContent: func() (bytes []byte, bytes2 []byte) { return nil, nil },
	}

	// used later  to filter the served resource by those that have expired.
	resourceExpirationEvaluator, err := genericapiserver.NewResourceExpirationEvaluator(*c.GenericConfig.Version)
	if err != nil {
		return nil, err
	}

    // 获取Service层
	apiGroupInfo := apiservicerest.NewRESTStorage(c.GenericConfig.MergedResourceConfig, c.GenericConfig.RESTOptionsGetter, resourceExpirationEvaluator.ShouldServeForVersion(1, 22))
    // 注册Service层到GenericAPIServer中
	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	enabledVersions := sets.NewString()
	for v := range apiGroupInfo.VersionedResourcesStorageMap {
		enabledVersions.Insert(v)
	}
	if !enabledVersions.Has(v1.SchemeGroupVersion.Version) {
		return nil, fmt.Errorf("API group/version %s must be enabled", v1.SchemeGroupVersion.String())
	}

	apisHandler := &apisHandler{
		codecs:         aggregatorscheme.Codecs,
		lister:         s.lister,
		discoveryGroup: discoveryGroup(enabledVersions),
	}
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", apisHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle("/apis/", apisHandler)

	apiserviceRegistrationController := NewAPIServiceRegistrationController(informerFactory.Apiregistration().V1().APIServices(), s)
	if len(c.ExtraConfig.ProxyClientCertFile) > 0 && len(c.ExtraConfig.ProxyClientKeyFile) > 0 {
		aggregatorProxyCerts, err := dynamiccertificates.NewDynamicServingContentFromFiles("aggregator-proxy-cert", c.ExtraConfig.ProxyClientCertFile, c.ExtraConfig.ProxyClientKeyFile)
		if err != nil {
			return nil, err
		}
		if err := aggregatorProxyCerts.RunOnce(); err != nil {
			return nil, err
		}
		aggregatorProxyCerts.AddListener(apiserviceRegistrationController)
		s.proxyCurrentCertKeyContent = aggregatorProxyCerts.CurrentCertKeyContent

		s.GenericAPIServer.AddPostStartHookOrDie("aggregator-reload-proxy-client-cert", func(context genericapiserver.PostStartHookContext) error {
			go aggregatorProxyCerts.Run(1, context.StopCh)
			return nil
		})
	}

	availableController, err := statuscontrollers.NewAvailableConditionController(
		informerFactory.Apiregistration().V1().APIServices(),
		c.GenericConfig.SharedInformerFactory.Core().V1().Services(),
		c.GenericConfig.SharedInformerFactory.Core().V1().Endpoints(),
		apiregistrationClient.ApiregistrationV1(),
		c.ExtraConfig.ProxyTransport,
		(func() ([]byte, []byte))(s.proxyCurrentCertKeyContent),
		s.serviceResolver,
		c.GenericConfig.EgressSelector,
	)
	if err != nil {
		return nil, err
	}

	s.GenericAPIServer.AddPostStartHookOrDie("start-kube-aggregator-informers", func(context genericapiserver.PostStartHookContext) error {
		informerFactory.Start(context.StopCh)
		c.GenericConfig.SharedInformerFactory.Start(context.StopCh)
		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-registration-controller", func(context genericapiserver.PostStartHookContext) error {
		go apiserviceRegistrationController.Run(context.StopCh, apiServiceRegistrationControllerInitiated)
		select {
		case <-context.StopCh:
		case <-apiServiceRegistrationControllerInitiated:
		}

		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("apiservice-status-available-controller", func(context genericapiserver.PostStartHookContext) error {
		// if we end up blocking for long periods of time, we may need to increase workers.
		go availableController.Run(5, context.StopCh)
		return nil
	})

	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) {
		// Spawn a goroutine in aggregator apiserver to update storage version for
		// all built-in resources
		s.GenericAPIServer.AddPostStartHookOrDie(StorageVersionPostStartHookName, func(hookContext genericapiserver.PostStartHookContext) error {
			// Wait for apiserver-identity to exist first before updating storage
			// versions, to avoid storage version GC accidentally garbage-collecting
			// storage versions.
			kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
			if err != nil {
				return err
			}
			if err := wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
				_, err := kubeClient.CoordinationV1().Leases(metav1.NamespaceSystem).Get(
					context.TODO(), s.GenericAPIServer.APIServerID, metav1.GetOptions{})
				if apierrors.IsNotFound(err) {
					return false, nil
				}
				if err != nil {
					return false, err
				}
				return true, nil
			}, hookContext.StopCh); err != nil {
				return fmt.Errorf("failed to wait for apiserver-identity lease %s to be created: %v",
					s.GenericAPIServer.APIServerID, err)
			}
			// Technically an apiserver only needs to update storage version once during bootstrap.
			// Reconcile StorageVersion objects every 10 minutes will help in the case that the
			// StorageVersion objects get accidentally modified/deleted by a different agent. In that
			// case, the reconciliation ensures future storage migration still works. If nothing gets
			// changed, the reconciliation update is a noop and gets short-circuited by the apiserver,
			// therefore won't change the resource version and trigger storage migration.
			go wait.PollImmediateUntil(10*time.Minute, func() (bool, error) {
				// All apiservers (aggregator-apiserver, kube-apiserver, apiextensions-apiserver)
				// share the same generic apiserver config. The same StorageVersion manager is used
				// to register all built-in resources when the generic apiservers install APIs.
				s.GenericAPIServer.StorageVersionManager.UpdateStorageVersions(hookContext.LoopbackClientConfig, s.GenericAPIServer.APIServerID)
				return false, nil
			}, hookContext.StopCh)
			// Once the storage version updater finishes the first round of update,
			// the PostStartHook will return to unblock /healthz. The handler chain
			// won't block write requests anymore. Check every second since it's not
			// expensive.
			wait.PollImmediateUntil(1*time.Second, func() (bool, error) {
				return s.GenericAPIServer.StorageVersionManager.Completed(), nil
			}, hookContext.StopCh)
			return nil
		})
	}

	return s, nil
}
```