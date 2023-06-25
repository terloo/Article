# Factory
执行每个kubectl之前，都要实例化一个cmdutil.Factory接口的对象。Factory是一个通用对象，提供了与api-server的交互方式，以及验证对象的方法。

## 接口结构
```go
// vendor/k8s.io/kubectl/pkg/cmd/util/factory.go
type Factory interface {
	genericclioptions.RESTClientGetter
	DynamicClient() (dynamic.Interface, error)
	KubernetesClientSet() (*kubernetes.Clientset, error)
	RESTClient() (*restclient.RESTClient, error)
	NewBuilder() *resource.Builder
	Validator(validate bool) (validation.Schema, error)

	// Returns a RESTClient for working with the specified RESTMapping or an error. This is intended
	// for working with arbitrary resources and is not guaranteed to point to a Kubernetes APIServer.
	ClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error)
	// Returns a RESTClient for working with Unstructured objects.
	UnstructuredClientForMapping(mapping *meta.RESTMapping) (resource.RESTClient, error)

	// OpenAPISchema returns the parsed openapi schema definition
	OpenAPISchema() (openapi.Resources, error)
	// OpenAPIGetter returns a getter for the openapi schema document
	OpenAPIGetter() discovery.OpenAPISchemaInterface
}
```
1. genericclioptions.RESTClientGetter：生成客户端配置相关接口
2. DynamicClient：动态客户端
3. KubernetesClientSet：ClientSet客户端
4. RESTClient：REST客户端
5. NewBuilder：实例化Builder，Builder用于将命令行的参数转化为资源对象
6. Validator：验证资源对象

## 实现类
// vendor/k8s.io/kubectl/pkg/cmd/factory_client_access.go
```go
type factoryImpl struct {
	clientGetter genericclioptions.RESTClientGetter

	// Caches OpenAPI document and parsed resources
    // 用于缓存OpenAPI文档
	openAPIParser *openapi.CachedOpenAPIParser
	openAPIGetter *openapi.CachedOpenAPIGetter
	parser        sync.Once
	getter        sync.Once
}

// 传入一个用于解析客户端配置的RESTClientGetter，直接实例化
func NewFactory(clientGetter genericclioptions.RESTClientGetter) Factory {
	if clientGetter == nil {
		panic("attempt to instantiate client_access_factory with nil clientGetter")
	}
	f := &factoryImpl{
		clientGetter: clientGetter,
	}

	return f
}
```