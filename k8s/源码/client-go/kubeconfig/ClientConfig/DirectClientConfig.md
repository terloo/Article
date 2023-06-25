# DirectClientConfig
实现了由kubeconfig生成restclient.Config的基础逻辑

## 结构体
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
type DirectClientConfig struct {
	config         clientcmdapi.Config
	contextName    string
	overrides      *ConfigOverrides
	fallbackReader io.Reader
	configAccess   ConfigAccess
	// promptedCredentials store the credentials input by the user
	promptedCredentials promptedCredentials
}
```
1. config：由ClientConfigLoader加载得到的kubeconfig类
2. contextName：当前使用的context
3. overrides：自定义覆盖配置
4. configAccess：用于访问kubeconfig文件的类，此类中使用此字段完成kueconfig配置改动持久化
5. fallbackReader：在kubeconfig无法验证用户时，读取BasicAuth需要使用的账号密码
6. promptedCredentials：BasicAuth方式验证所使用账号密码

## 构造函数
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
// NewDefaultClientConfig creates a DirectClientConfig using the config.CurrentContext as the context name
// 使用kubeconfig中CurrentContext作为使用的Context，使用默认的ClientConfigLoadingRules读取规则作为configAccess
func NewDefaultClientConfig(config clientcmdapi.Config, overrides *ConfigOverrides) ClientConfig {
	return &DirectClientConfig{config, config.CurrentContext, overrides, nil, NewDefaultClientConfigLoadingRules(), promptedCredentials{}}
}

// NewNonInteractiveClientConfig creates a DirectClientConfig using the passed context name and does not have a fallback reader for auth information
func NewNonInteractiveClientConfig(config clientcmdapi.Config, contextName string, overrides *ConfigOverrides, configAccess ConfigAccess) ClientConfig {
	return &DirectClientConfig{config, contextName, overrides, nil, configAccess, promptedCredentials{}}
}

// NewInteractiveClientConfig creates a DirectClientConfig using the passed context name and a reader in case auth information is not provided via files or flags
func NewInteractiveClientConfig(config clientcmdapi.Config, contextName string, overrides *ConfigOverrides, fallbackReader io.Reader, configAccess ConfigAccess) ClientConfig {
	return &DirectClientConfig{config, contextName, overrides, fallbackReader, configAccess, promptedCredentials{}}
}
```

## ClientConfig
由kubeconfig生成restclient.Config的基础逻辑
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
func (config *DirectClientConfig) ClientConfig() (*restclient.Config, error) {
	// check that getAuthInfo, getContext, and getCluster do not return an error.
	// Do this before checking if the current config is usable in the event that an
	// AuthInfo, Context, or Cluster config with user-defined names are not found.
	// This provides a user with the immediate cause for error if one is found

    // 从config字段中读取authInfo Context Cluster三大信息
	// getAuthInfo，getCluster，getContext这些方法会使用override覆盖原始配置
	configAuthInfo, err := config.getAuthInfo()
	if err != nil {
		return nil, err
	}

	_, err = config.getContext()
	if err != nil {
		return nil, err
	}

	configClusterInfo, err := config.getCluster()
	if err != nil {
		return nil, err
	}

	// 验证构建时传入的kubeconfig类可用性，排除nil或者空配置或其他不合法情况
	if err := config.ConfirmUsable(); err != nil {
		return nil, err
	}

    // 新生成一个restclient.Config，然后向其中填充字段
	clientConfig := &restclient.Config{}
	clientConfig.Host = configClusterInfo.Server
	if configClusterInfo.ProxyURL != "" {
		u, err := parseProxyURL(configClusterInfo.ProxyURL)
		if err != nil {
			return nil, err
		}
		clientConfig.Proxy = http.ProxyURL(u)
	}

    // 使用overrides的超时时间覆盖kubeconfig中的超时时间
	if config.overrides != nil && len(config.overrides.Timeout) > 0 {
		timeout, err := ParseTimeout(config.overrides.Timeout)
		if err != nil {
			return nil, err
		}
		clientConfig.Timeout = timeout
	}

	if u, err := url.ParseRequestURI(clientConfig.Host); err == nil && u.Opaque == "" && len(u.Path) > 1 {
		u.RawQuery = ""
		u.Fragment = ""
		clientConfig.Host = u.String()
	}
	if len(configAuthInfo.Impersonate) > 0 {
		clientConfig.Impersonate = restclient.ImpersonationConfig{
			UserName: configAuthInfo.Impersonate,
			UID:      configAuthInfo.ImpersonateUID,
			Groups:   configAuthInfo.ImpersonateGroups,
			Extra:    configAuthInfo.ImpersonateUserExtra,
		}
	}

	// only try to read the auth information if we are secure
	if restclient.IsConfigTransportTLS(*clientConfig) {
		var err error
		var persister restclient.AuthProviderConfigPersister
		if config.configAccess != nil {
			authInfoName, _ := config.getAuthInfoName()
			persister = PersisterForUser(config.configAccess, authInfoName)
		}
		userAuthPartialConfig, err := config.getUserIdentificationPartialConfig(configAuthInfo, config.fallbackReader, persister, configClusterInfo)
		if err != nil {
			return nil, err
		}
		mergo.Merge(clientConfig, userAuthPartialConfig, mergo.WithOverride)

		serverAuthPartialConfig, err := getServerIdentificationPartialConfig(configAuthInfo, configClusterInfo)
		if err != nil {
			return nil, err
		}
		mergo.Merge(clientConfig, serverAuthPartialConfig, mergo.WithOverride)
	}

	return clientConfig, nil
}
```