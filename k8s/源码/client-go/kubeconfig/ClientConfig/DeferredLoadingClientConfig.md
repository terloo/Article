# DeferredLoadingClientConfig
延迟加载实现，在调用ClientConfig()方法时，才会加载文件。如果无法从loader中获取配置，则获取集群配置。

## 结构体
```go
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
type DeferredLoadingClientConfig struct {
	loader         ClientConfigLoader
	overrides      *ConfigOverrides
	fallbackReader io.Reader

	clientConfig ClientConfig
	loadingLock  sync.Mutex

	// provided for testing
	icc InClusterConfig
}
```
1. loader：配置加载合并器
2. overrides：配置覆盖
3. fallbackReader：在kubeconfig无法验证用户时，读取BasicAuth需要使用的账号密码
4. clientConfig：缓存构建的DirectClinetConfig
5. icc：集群配置生成器，如果无法获取配置，则使用集群配置

## 构造函数
```go
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
func NewNonInteractiveDeferredLoadingClientConfig(loader ClientConfigLoader, overrides *ConfigOverrides) ClientConfig {
	return &DeferredLoadingClientConfig{loader: loader, overrides: overrides, icc: &inClusterClientConfig{overrides: overrides}}
}

func NewInteractiveDeferredLoadingClientConfig(loader ClientConfigLoader, overrides *ConfigOverrides, fallbackReader io.Reader) ClientConfig {
	return &DeferredLoadingClientConfig{loader: loader, overrides: overrides, icc: &inClusterClientConfig{overrides: overrides}, fallbackReader: fallbackReader}
}
```

## createClientConfig
从loader生成最新的已合并kubeconfig，再构建DirectClientConfig(并缓存)
```go
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
func (config *DeferredLoadingClientConfig) createClientConfig() (ClientConfig, error) {
	config.loadingLock.Lock()
	defer config.loadingLock.Unlock()

	if config.clientConfig != nil {
		return config.clientConfig, nil
	}
	mergedConfig, err := config.loader.Load()
	if err != nil {
		return nil, err
	}

	var currentContext string
	if config.overrides != nil {
		currentContext = config.overrides.CurrentContext
	}
	if config.fallbackReader != nil {
		config.clientConfig = NewInteractiveClientConfig(*mergedConfig, currentContext, config.overrides, config.fallbackReader, config.loader)
	} else {
		config.clientConfig = NewNonInteractiveClientConfig(*mergedConfig, currentContext, config.overrides, config.loader)
	}
	return config.clientConfig, nil
}
```

## RawConfig
```go
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
func (config *DeferredLoadingClientConfig) RawConfig() (clientcmdapi.Config, error) {
	mergedConfig, err := config.createClientConfig()
	if err != nil {
		return clientcmdapi.Config{}, err
	}

	return mergedConfig.RawConfig()
}
```

## ClientConfig
构建DirectConfig，再从DirectConfig中获取restclient
```go
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
func (config *DeferredLoadingClientConfig) ClientConfig() (*restclient.Config, error) {
	// 构建DirectConfig
	mergedClientConfig, err := config.createClientConfig()
	if err != nil {
		return nil, err
	}

	// load the configuration and return on non-empty errors and if the
	// content differs from the default config
	// 获取restclient.Config
	mergedConfig, err := mergedClientConfig.ClientConfig()
	switch {
	case err != nil:
		if !IsEmptyConfig(err) {
			// 如果不是空配置错误，则返回错误，否则先不返回
			// return on any error except empty config
			return nil, err
		}
	case mergedConfig != nil:
		// the configuration is valid, but if this is equal to the defaults we should try
		// in-cluster configuration
		// 如果配置还是默认配置没发生改变，则先不返回
		if !config.loader.IsDefaultConfig(mergedConfig) {
			return mergedConfig, nil
		}
	}

	// check for in-cluster configuration and use it
	// 检查是否在集群中，如果在集群中，使用集群Pod中的配置
	if config.icc.Possible() {
		klog.V(4).Infof("Using in-cluster configuration")
		return config.icc.ClientConfig()
	}

	// return the result of the merged client config
	// 返回值    1 默认配置,nil  2  nil,非nil的error
	return mergedConfig, err
}
```