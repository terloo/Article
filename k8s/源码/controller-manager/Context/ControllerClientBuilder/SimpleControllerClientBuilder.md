# SimpleControllerClientBuilder
简单的实现类，使用其ClientConfig字段创建clientset

## 实现类
```go
// k8s.io/controller-manager/pkg/clientbuilder/client_builder.go
type SimpleControllerClientBuilder struct {
	// ClientConfig is a skeleton config to clone and use as the basis for each controller client
	ClientConfig *restclient.Config
}
```

## 方法
```go
// k8s.io/controller-manager/pkg/clientbuilder/client_builder.go
// Config returns a client config for a fixed client
func (b SimpleControllerClientBuilder) Config(name string) (*restclient.Config, error) {
	clientConfig := *b.ClientConfig
	return restclient.AddUserAgent(&clientConfig, name), nil
}

// ConfigOrDie returns a client config if no error from previous config func.
// If it gets an error getting the client, it will log the error and kill the process it's running in.
func (b SimpleControllerClientBuilder) ConfigOrDie(name string) *restclient.Config {
	clientConfig, err := b.Config(name)
	if err != nil {
		klog.Fatal(err)
	}
	return clientConfig
}

// Client returns a clientset.Interface built from the ClientBuilder
func (b SimpleControllerClientBuilder) Client(name string) (clientset.Interface, error) {
	clientConfig, err := b.Config(name)
	if err != nil {
		return nil, err
	}
	return clientset.NewForConfig(clientConfig)
}

// ClientOrDie returns a clientset.interface built from the ClientBuilder with no error.
// If it gets an error getting the client, it will log the error and kill the process it's running in.
func (b SimpleControllerClientBuilder) ClientOrDie(name string) clientset.Interface {
	client, err := b.Client(name)
	if err != nil {
		klog.Fatal(err)
	}
	return client
}

// DiscoveryClientOrDie returns a discovery.DiscoveryInterface built from the ClientBuilder
// Discovery is special because it will artificially pump the burst quite high to handle the many discovery requests.
func (b SimpleControllerClientBuilder) DiscoveryClient(name string) (discovery.DiscoveryInterface, error) {
	clientConfig, err := b.Config(name)
	if err != nil {
		return nil, err
	}
	// Discovery makes a lot of requests infrequently.  This allows the burst to succeed and refill to happen
	// in just a few seconds.
	clientConfig.Burst = 200
	clientConfig.QPS = 20
	return clientset.NewForConfig(clientConfig)
}

// DiscoveryClientOrDie returns a discovery.DiscoveryInterface built from the ClientBuilder with no error.
// Discovery is special because it will artificially pump the burst quite high to handle the many discovery requests.
// If it gets an error getting the client, it will log the error and kill the process it's running in.
func (b SimpleControllerClientBuilder) DiscoveryClientOrDie(name string) discovery.DiscoveryInterface {
	client, err := b.DiscoveryClient(name)
	if err != nil {
		klog.Fatal(err)
	}
	return client
}
```