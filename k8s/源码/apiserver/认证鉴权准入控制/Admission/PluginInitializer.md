# PluginInitializer
PluginInitializer是用于对Plugin进行初始化的接口，即给plugin中的可共享的资源字段赋值

## 接口
```go
// vendor/k8s.io/apiserver/pkg/admission/interfaces.go
type PluginInitializer interface {
	Initialize(plugin Interface)
}
```

## 实现类
根据Plugin实现的接口，来决定是否调用这些已实现的接口，传入相应的资源

### webhook
对webhook类型的Plugin进行初始化
```go
// vendor/k8s.io/apiserver/pkg/admission/plugin/initializer/initializer.go
type PluginInitializer struct {
    // service资源解析器
	serviceResolver                   webhook.ServiceResolver
    // 验证信息包装器
	authenticationInfoResolverWrapper webhook.AuthenticationInfoResolverWrapper
}

// Plugin实现该接口说明需要初始化器传入一个ServiceResolver
type WantsServiceResolver interface {
	SetServiceResolver(webhook.ServiceResolver)
}

// Plugin实现该接口说明需要初始化器传入一个AuthenticationInfoResolverWrapper
type WantsAuthenticationInfoResolverWrapper interface {
	SetAuthenticationInfoResolverWrapper(wrapper webhook.AuthenticationInfoResolverWrapper)
	admission.InitializationValidator
}

func (i *PluginInitializer) Initialize(plugin admission.Interface) {
    // 判断plugin是否实现了某接口，如果实现了，调用该接口传入相应的资源
	if wants, ok := plugin.(WantsServiceResolver); ok {
		wants.SetServiceResolver(i.serviceResolver)
	}

	if wants, ok := plugin.(WantsAuthenticationInfoResolverWrapper); ok {
		if i.authenticationInfoResolverWrapper != nil {
			wants.SetAuthenticationInfoResolverWrapper(i.authenticationInfoResolverWrapper)
		}
	}
}
```

### kubePlugin
对k8s内部使用Plugin进行初始化
```go
// pkg/kubeapiserver/admission/initializer.go
type PluginInitializer struct {
	cloudConfig        []byte
	restMapper         meta.RESTMapper
	quotaConfiguration quota.Configuration
}

// Plugin实现该接口说明需要初始化器传入一个CloudConfig
type WantsCloudConfig interface {
	SetCloudConfig([]byte)
}

// Plugin实现该接口说明需要初始化器传入一个RESTMapper
type WantsRESTMapper interface {
	SetRESTMapper(meta.RESTMapper)
}

func (i *PluginInitializer) Initialize(plugin admission.Interface) {
	if wants, ok := plugin.(WantsCloudConfig); ok {
		wants.SetCloudConfig(i.cloudConfig)
	}

	if wants, ok := plugin.(WantsRESTMapper); ok {
		wants.SetRESTMapper(i.restMapper)
	}

	if wants, ok := plugin.(initializer.WantsQuotaConfiguration); ok {
		wants.SetQuotaConfiguration(i.quotaConfiguration)
	}
}
```

### generic
对所有Plugin进行初始化
```go
// vendor/k8s.io/apiserver/pkg/admission/initializer/initializer.go
type pluginInitializer struct {
	externalClient    kubernetes.Interface
	externalInformers informers.SharedInformerFactory
	authorizer        authorizer.Authorizer
	featureGates      featuregate.FeatureGate
}

func (i pluginInitializer) Initialize(plugin admission.Interface) {
	// First tell the plugin about enabled features, so it can decide whether to start informers or not
	if wants, ok := plugin.(WantsFeatures); ok {
		wants.InspectFeatureGates(i.featureGates)
	}

	if wants, ok := plugin.(WantsExternalKubeClientSet); ok {
		wants.SetExternalKubeClientSet(i.externalClient)
	}

	if wants, ok := plugin.(WantsExternalKubeInformerFactory); ok {
		wants.SetExternalKubeInformerFactory(i.externalInformers)
	}

	if wants, ok := plugin.(WantsAuthorizer); ok {
		wants.SetAuthorizer(i.authorizer)
	}
}

// vendor/k8s.io/apiserver/pkg/admission/initializer/interfaces.go
// Plugin实现该接口说明需要初始化器传入一个kubernetes.Interface(k8s客户端)
type WantsExternalKubeClientSet interface {
	SetExternalKubeClientSet(kubernetes.Interface)
	admission.InitializationValidator
}

// Plugin实现该接口说明需要初始化器传入一个SharedInformerFactory
type WantsExternalKubeInformerFactory interface {
	SetExternalKubeInformerFactory(informers.SharedInformerFactory)
	admission.InitializationValidator
}

// Plugin实现该接口说明需要初始化器传入一个Authorizer
type WantsAuthorizer interface {
	SetAuthorizer(authorizer.Authorizer)
	admission.InitializationValidator
}

// Plugin实现该接口说明需要初始化器传入一个Configuration
type WantsQuotaConfiguration interface {
	SetQuotaConfiguration(quota.Configuration)
	admission.InitializationValidator
}

// Plugin实现该接口说明需要初始化器传入一个FeatureGate
// 使用FeatureGate不应该直接保存FeatureGate，而应该保存features.Enabled("XXX")的结果
type WantsFeatures interface {
	InspectFeatureGates(featuregate.FeatureGate)
	admission.InitializationValidator
}
```

### PluginInitializers整合多个Initializer
```go
// vendor/k8s.io/apiserver/pkg/admission/plugins.go
type PluginInitializers []PluginInitializer

func (pp PluginInitializers) Initialize(plugin Interface) {
	for _, p := range pp {
		p.Initialize(plugin)
	}
}
```