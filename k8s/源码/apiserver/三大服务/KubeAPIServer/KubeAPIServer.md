# KubeAPIServer

## 所含接口分类
1. Legacy：k8s早期设计的API，即核心API，没有分组概念。统一前缀为`/api/v1`
2. 分组API：带有分组的API，统一前缀为`/apis/<group>/<version>`

## 结构体
```go
// pkg/controlplane/instance.go
type Instance struct {
    // GenericAPIServer
	GenericAPIServer *genericapiserver.GenericAPIServer

	ClusterAuthenticationInfo clusterauthenticationtrust.ClusterAuthenticationInfo
}
```

## New
```go
// pkg/controlplane/instance.go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	if reflect.DeepEqual(c.ExtraConfig.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
		return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
	}

    // 先由GenriceConfig构建一个GenericAPIServer，名字是kube-apiserver
	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

    // 如果需要，注册logs接口
	if c.ExtraConfig.EnableLogsSupport {
		routes.Logs{}.Install(s.Handler.GoRestfulContainer)
	}

	// Metadata and keys are expected to only change across restarts at present,
	// so we just marshal immediately and serve the cached JSON bytes.
	md, err := serviceaccount.NewOpenIDMetadata(
		c.ExtraConfig.ServiceAccountIssuerURL,
		c.ExtraConfig.ServiceAccountJWKSURI,
		c.GenericConfig.ExternalAddress,
		c.ExtraConfig.ServiceAccountPublicKeys,
	)
	if err != nil {
		// If there was an error, skip installing the endpoints and log the
		// error, but continue on. We don't return the error because the
		// metadata responses require additional, backwards incompatible
		// validation of command-line options.
		msg := fmt.Sprintf("Could not construct pre-rendered responses for"+
			" ServiceAccountIssuerDiscovery endpoints. Endpoints will not be"+
			" enabled. Error: %v", err)
		if c.ExtraConfig.ServiceAccountIssuerURL != "" {
			// The user likely expects this feature to be enabled if issuer URL is
			// set and the feature gate is enabled. In the future, if there is no
			// longer a feature gate and issuer URL is not set, the user may not
			// expect this feature to be enabled. We log the former case as an Error
			// and the latter case as an Info.
			klog.Error(msg)
		} else {
			klog.Info(msg)
		}
	} else {
		routes.NewOpenIDMetadataServer(md.ConfigJSON, md.PublicKeysetJSON).
			Install(s.Handler.GoRestfulContainer)
	}

    // 新建KubeAPIServer
	m := &Instance{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}

	// install legacy rest storage
	if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
		legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
			StorageFactory:              c.ExtraConfig.StorageFactory,
			ProxyTransport:              c.ExtraConfig.ProxyTransport,
			KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
			EventTTL:                    c.ExtraConfig.EventTTL,
			ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
			SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
			ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
			LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
			ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
			ExtendExpiration:            c.ExtraConfig.ExtendExpiration,
			ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
			APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
		}
        // 注册legacyAPI
		if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider); err != nil {
			return nil, err
		}
	}

	// The order here is preserved in discovery.
	// If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
	// the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
	// This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
	// with specific priorities.
	// TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
	// handlers that we have.
	restStorageProviders := []RESTStorageProvider{
		apiserverinternalrest.StorageProvider{},
		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
		autoscalingrest.RESTStorageProvider{},
		batchrest.RESTStorageProvider{},
		certificatesrest.RESTStorageProvider{},
		coordinationrest.RESTStorageProvider{},
		discoveryrest.StorageProvider{},
		networkingrest.RESTStorageProvider{},
		noderest.RESTStorageProvider{},
		policyrest.RESTStorageProvider{},
		rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
		schedulingrest.RESTStorageProvider{},
		storagerest.RESTStorageProvider{},
		flowcontrolrest.RESTStorageProvider{},
		// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
		// See https://github.com/kubernetes/kubernetes/issues/42392
		appsrest.StorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}
    // 注册分组API
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}

    // 添加一个PostStartHook，名为start-cluster-authentication-info-controller
	m.GenericAPIServer.AddPostStartHookOrDie("start-cluster-authentication-info-controller", func(hookContext genericapiserver.PostStartHookContext) error {
		kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
		if err != nil {
			return err
		}
		controller := clusterauthenticationtrust.NewClusterAuthenticationTrustController(m.ClusterAuthenticationInfo, kubeClient)

		// prime values and start listeners
		if m.ClusterAuthenticationInfo.ClientCA != nil {
			m.ClusterAuthenticationInfo.ClientCA.AddListener(controller)
			if controller, ok := m.ClusterAuthenticationInfo.ClientCA.(dynamiccertificates.ControllerRunner); ok {
				// runonce to be sure that we have a value.
				if err := controller.RunOnce(); err != nil {
					runtime.HandleError(err)
				}
				go controller.Run(1, hookContext.StopCh)
			}
		}
		if m.ClusterAuthenticationInfo.RequestHeaderCA != nil {
			m.ClusterAuthenticationInfo.RequestHeaderCA.AddListener(controller)
			if controller, ok := m.ClusterAuthenticationInfo.RequestHeaderCA.(dynamiccertificates.ControllerRunner); ok {
				// runonce to be sure that we have a value.
				if err := controller.RunOnce(); err != nil {
					runtime.HandleError(err)
				}
				go controller.Run(1, hookContext.StopCh)
			}
		}

		go controller.Run(1, hookContext.StopCh)
		return nil
	})

	if utilfeature.DefaultFeatureGate.Enabled(apiserverfeatures.APIServerIdentity) {
        // 如果启用了Feature APIServerIdentity则添加AddPostStartHook，名为start-kube-apiserver-identity-lease-controller和start-kube-apiserver-identity-lease-garbage-collector
		m.GenericAPIServer.AddPostStartHookOrDie("start-kube-apiserver-identity-lease-controller", func(hookContext genericapiserver.PostStartHookContext) error {
			kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
			if err != nil {
				return err
			}
			controller := lease.NewController(
				clock.RealClock{},
				kubeClient,
				m.GenericAPIServer.APIServerID,
				int32(c.ExtraConfig.IdentityLeaseDurationSeconds),
				nil,
				time.Duration(c.ExtraConfig.IdentityLeaseRenewIntervalSeconds)*time.Second,
				metav1.NamespaceSystem,
				labelAPIServerHeartbeat)
			go controller.Run(wait.NeverStop)
			return nil
		})
		m.GenericAPIServer.AddPostStartHookOrDie("start-kube-apiserver-identity-lease-garbage-collector", func(hookContext genericapiserver.PostStartHookContext) error {
			kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
			if err != nil {
				return err
			}
			go apiserverleasegc.NewAPIServerLeaseGC(
				kubeClient,
				time.Duration(c.ExtraConfig.IdentityLeaseDurationSeconds)*time.Second,
				metav1.NamespaceSystem,
				KubeAPIServerIdentityLeaseLabelSelector,
			).Run(wait.NeverStop)
			return nil
		})
	}

	return m, nil
}
```