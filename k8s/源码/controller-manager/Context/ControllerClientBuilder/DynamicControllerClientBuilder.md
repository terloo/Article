# DynamicControllerClientBuilder
动态ControllerClientBuilder，

## 实现类
```go
// k8s.io/controller-manager/pkg/clientbuilder/client_builder_dynamic.go
type DynamicControllerClientBuilder struct {
	// ClientConfig is a skeleton config to clone and use as the basis for each controller client
	ClientConfig *restclient.Config

	// CoreClient is used to provision service accounts if needed and watch for their associated tokens
	// to construct a controller client
	CoreClient v1core.CoreV1Interface

	// Namespace is the namespace used to host the service accounts that will back the
	// controllers.  It must be highly privileged namespace which normal users cannot inspect.
	Namespace string

	// roundTripperFuncMap is a cache stores the corresponding roundtripper func for each
	// service account
	roundTripperFuncMap map[string]func(http.RoundTripper) http.RoundTripper

	// expirationSeconds defines the token expiration seconds
	expirationSeconds int64

	// leewayPercent defines the percentage of expiration left before the client trigger a token rotation.
	leewayPercent int

	mutex sync.Mutex

	clock clock.Clock
}
```
1. ClientConfig：restclient.Config
2. CoreClient：corev1的客户端
3. Namespace：k8s系统使用的命名空间，kube-system

## 构建函数
```go
// k8s.io/controller-manager/pkg/clientbuilder/client_builder_dynamic.go
func NewDynamicClientBuilder(clientConfig *restclient.Config, coreClient v1core.CoreV1Interface, ns string) ControllerClientBuilder {
	builder := &DynamicControllerClientBuilder{
		ClientConfig:        clientConfig,
		CoreClient:          coreClient,
		Namespace:           ns,
		roundTripperFuncMap: map[string]func(http.RoundTripper) http.RoundTripper{},
		expirationSeconds:   defaultExpirationSeconds,
		leewayPercent:       defaultLeewayPercent,
		clock:               clock.RealClock{},
	}
	return builder
}
```