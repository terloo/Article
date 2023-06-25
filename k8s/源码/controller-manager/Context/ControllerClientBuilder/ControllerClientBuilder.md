# ControllerClientBuilder
ControllerClientBuilder是controller-manager中用于构建k8s-client的结构体

## 接口
```go
// k8s.io/controller-manager/pkg/clientbuilder/client_builder.go
type ControllerClientBuilder interface {
	Config(name string) (*restclient.Config, error)
	ConfigOrDie(name string) *restclient.Config
	Client(name string) (clientset.Interface, error)
	ClientOrDie(name string) clientset.Interface
	DiscoveryClient(name string) (discovery.DiscoveryInterface, error)
	DiscoveryClientOrDie(name string) discovery.DiscoveryInterface
}
```
1. name参数：用于标识该client的调用方
2. Config：获取client的config，如果错误则返回
3. ConfigOrDie：获取client的config，如果错误则退出程序
4. Client：直接获取clientset类型的客户端
5. DiscoveryClient：直接获取discovery类型的client

## 实现类
1. SimpleControllerClientBuilder：一个简单的restclient包装
2. DynamicControllerClientBuilder：配置config.WrapTransport字段，使用ServiceAccount来访问api-server

## createClientBuilder
```go
// cmd/kube-controller-manager/app/controllermanager.go
func createClientBuilders(c *config.CompletedConfig) (clientBuilder clientbuilder.ControllerClientBuilder, rootClientBuilder clientbuilder.ControllerClientBuilder) {
	// rootClientBuilder为SimpleControllerClientBuilder
	rootClientBuilder = clientbuilder.SimpleControllerClientBuilder{
		ClientConfig: c.Kubeconfig,
	}
	// 如果需要每个Controller使用不同的ServiceAccount，则使用NewDynamicClientBuilder作为clientBuilder
	if c.ComponentConfig.KubeCloudShared.UseServiceAccountCredentials {
		if len(c.ComponentConfig.SAController.ServiceAccountKeyFile) == 0 {
			// It's possible another controller process is creating the tokens for us.
			// If one isn't, we'll timeout and exit when our client builder is unable to create the tokens.
			klog.Warningf("--use-service-account-credentials was specified without providing a --service-account-private-key-file")
		}

		clientBuilder = clientbuilder.NewDynamicClientBuilder(
			// 去掉kubeconfig中身份验证的部分
			restclient.AnonymousClientConfig(c.Kubeconfig),
			c.Client.CoreV1(),
			metav1.NamespaceSystem)
	} else {
		// 否则clientBuilder和rootClientBuilder使用同一个builder即可
		clientBuilder = rootClientBuilder
	}
	return
}
```
