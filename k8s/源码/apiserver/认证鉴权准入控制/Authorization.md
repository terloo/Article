# Authorization
客户端发起的请求，只要有一个鉴权成功，即允许访问

## 初始化
1. 初始化默认命令行参数，有以下几种鉴权模式
   1. AlwaysAllow(默认启用)
   2. AlwaysDeny
   3. Webhook
   4. Node
   5. ABAB
   6. RBAC
2. 绑定Cobra命令行参数
3. 函数调用，目的为构建Authorizer和RuleResolver(Authorizer和RuleResolver一般是由同一个类进行实现)
4. 从BuiltInAuthorizationOptions生成authorizerConfig
5. 从authorizerConfig生成Authorizer和RuleResolver
```go
// 1. 初始化默认命令行参数
// cmd/kube-apiserver/app/options/options.go
func NewServerRunOptions() *ServerRunOptions {
	s := ServerRunOptions{
        // ...
        // 默认的mode为AlwaysAllow
		s.Authorization.AddFlags(fss.FlagSet("authorization"))
		// ...
	}
    // ...

	return &s
}

// 2. 绑定Cobra命令行参数
// cmd/kube-apiserver/app/options/options.go
func (s *ServerRunOptions) Flags() (fss cliflag.NamedFlagSets) {
	// ...
    // 字段Authentication是BuiltInAuthenticationOptions
	s.Authorization.AddFlags(fss.FlagSet("authorization"))
    // ...
}

// 3. 函数调用，目的为构建Authorizer和RuleResolver
// cmd/kube-apiserver/app/server.go
func BuildAuthorizer(s *options.ServerRunOptions, EgressSelector *egressselector.EgressSelector, versionedInformers clientgoinformers.SharedInformerFactory) (authorizer.Authorizer, authorizer.RuleResolver, error) {
    // 从BuiltInAuthorizationOptions生成authorizerConfig
	authorizationConfig := s.Authorization.ToAuthorizationConfig(versionedInformers)

	if EgressSelector != nil {
		egressDialer, err := EgressSelector.Lookup(egressselector.ControlPlane.AsNetworkContext())
		if err != nil {
			return nil, nil, err
		}
		authorizationConfig.CustomDial = egressDialer
	}

	return authorizationConfig.New()
}

// 4. 从BuiltInAuthorizationOptions生成authorizerConfig
// pkg/kubeapiserver/options/authorization.go
func (o *BuiltInAuthorizationOptions) ToAuthorizationConfig(versionedInformerFactory versionedinformers.SharedInformerFactory) authorizer.Config {
	return authorizer.Config{
		AuthorizationModes:          o.Modes,
		PolicyFile:                  o.PolicyFile,
		WebhookConfigFile:           o.WebhookConfigFile,
		WebhookVersion:              o.WebhookVersion,
		WebhookCacheAuthorizedTTL:   o.WebhookCacheAuthorizedTTL,
		WebhookCacheUnauthorizedTTL: o.WebhookCacheUnauthorizedTTL,
		VersionedInformerFactory:    versionedInformerFactory,
		WebhookRetryBackoff:         o.WebhookRetryBackoff,
	}
}


// 5. 从authorizerConfig生成Authorizer和RuleResolver
// pkg/kubeapiserver/authorizer/config.go
func (config Config) New() (authorizer.Authorizer, authorizer.RuleResolver, error) {
	if len(config.AuthorizationModes) == 0 {
		return nil, nil, fmt.Errorf("at least one authorization mode must be passed")
	}

	var (
		authorizers   []authorizer.Authorizer
		ruleResolvers []authorizer.RuleResolver
	)

	for _, authorizationMode := range config.AuthorizationModes {
		// Keep cases in sync with constant list in k8s.io/kubernetes/pkg/kubeapiserver/authorizer/modes/modes.go.
        // 根据使用的Mode来示例化相应的authorizer和ruleResolver
		switch authorizationMode {
		case modes.ModeNode:
			node.RegisterMetrics()
			graph := node.NewGraph()
			node.AddGraphEventHandlers(
				graph,
				config.VersionedInformerFactory.Core().V1().Nodes(),
				config.VersionedInformerFactory.Core().V1().Pods(),
				config.VersionedInformerFactory.Core().V1().PersistentVolumes(),
				config.VersionedInformerFactory.Storage().V1().VolumeAttachments(),
			)
			nodeAuthorizer := node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())
			authorizers = append(authorizers, nodeAuthorizer)
			ruleResolvers = append(ruleResolvers, nodeAuthorizer)

		case modes.ModeAlwaysAllow:
			alwaysAllowAuthorizer := authorizerfactory.NewAlwaysAllowAuthorizer()
			authorizers = append(authorizers, alwaysAllowAuthorizer)
			ruleResolvers = append(ruleResolvers, alwaysAllowAuthorizer)
		case modes.ModeAlwaysDeny:
			alwaysDenyAuthorizer := authorizerfactory.NewAlwaysDenyAuthorizer()
			authorizers = append(authorizers, alwaysDenyAuthorizer)
			ruleResolvers = append(ruleResolvers, alwaysDenyAuthorizer)
		case modes.ModeABAC:
			abacAuthorizer, err := abac.NewFromFile(config.PolicyFile)
			if err != nil {
				return nil, nil, err
			}
			authorizers = append(authorizers, abacAuthorizer)
			ruleResolvers = append(ruleResolvers, abacAuthorizer)
		case modes.ModeWebhook:
			if config.WebhookRetryBackoff == nil {
				return nil, nil, errors.New("retry backoff parameters for authorization webhook has not been specified")
			}
			webhookAuthorizer, err := webhook.New(config.WebhookConfigFile,
				config.WebhookVersion,
				config.WebhookCacheAuthorizedTTL,
				config.WebhookCacheUnauthorizedTTL,
				*config.WebhookRetryBackoff,
				config.CustomDial)
			if err != nil {
				return nil, nil, err
			}
			authorizers = append(authorizers, webhookAuthorizer)
			ruleResolvers = append(ruleResolvers, webhookAuthorizer)
		case modes.ModeRBAC:
			rbacAuthorizer := rbac.New(
				&rbac.RoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().Roles().Lister()},
				&rbac.RoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().RoleBindings().Lister()},
				&rbac.ClusterRoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoles().Lister()},
				&rbac.ClusterRoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoleBindings().Lister()},
			)
			authorizers = append(authorizers, rbacAuthorizer)
			ruleResolvers = append(ruleResolvers, rbacAuthorizer)
		default:
			return nil, nil, fmt.Errorf("unknown authorization mode %s specified", authorizationMode)
		}
	}

    // 聚合为一个
	return union.New(authorizers...), union.NewRuleResolvers(ruleResolvers...), nil
}

// 6. 聚合authorizer和ruleResolver为一个，在鉴权时执行所有的authorizer和ruleResolver
```