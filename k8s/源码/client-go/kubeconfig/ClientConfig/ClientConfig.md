# ClientConfig
集中常用的构建restclient.Config的接口

## 实现类
1. DirectClientConfig：构建时直接传入`clientcmdapi.Config`类，执行ClientConfig方法时，检验并使用传入的`clientcmdapi.Config`生成restclient.Config
2. DeferredLoadingClientConfig：构建时传入一个`ConfigAccess`接口的实现类作为loader。执行ClientConfig方法时，使用loader生成`clientcmdapi.Config`，再使用`clientcmdapi.Config`生成DirectClientConfig，最终构建restclient.Config。如果配置为空，将会尝试生成集群配置
3. clientConfig：其他ClientConfig实现类的包装类，当配置为空时，会返回更友好的错误信息
4. inClusterClientConfig：InClusterConfig接口的实现类。当程序在集群中时，使用Pod中的信息来生成restclient.Config

## 接口
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
type ClientConfig interface {
	// RawConfig returns the merged result of all overrides
	RawConfig() (clientcmdapi.Config, error)
	// ClientConfig returns a complete client config
	ClientConfig() (*restclient.Config, error)
	// Namespace returns the namespace resulting from the merged
	// result of all overrides and a boolean indicating if it was
	// overridden
	Namespace() (string, bool, error)
	// ConfigAccess returns the rules for loading/persisting the config.
	ConfigAccess() ConfigAccess
}
```
1. RawConfig：返回所有配置来源覆盖合并后的kubeconfig，类型为k8s.io/client-go/tools/clientcmd/api.Config
2. ClientConfig：返回由Raw生成的RESTClient的config，类型为k8s.io/client-go/rest/config.Config
3. Namespace：返回未指定Namespace时的默认命名空间
4. ConfigAccess：返回该ClientConfig用于加载配置文件的类

## InClusterConfig
ClientConfig的子接口，新加了一个用于判断是否在Pod中的接口
```go
// k8s.io/client-go/tools/clientcmd/merged_client_builder.go
type InClusterConfig interface {
	ClientConfig
	Possible() bool
}
```
1. Possible：用于判断调用方是否运行在Pod中
