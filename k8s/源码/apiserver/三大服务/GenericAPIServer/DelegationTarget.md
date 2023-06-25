# DelegationTarget
GenricAPIServer是DelegationTarget接口的实现类，亦是GenricAPIServer接口体中的一个字段。
创建Aggregator、KubeAPIServer、APIExtensionsServer三个服务时都需要传入一个DelegationTarget(即GenericAPIServer)，  
以便在本服务的GenericAPIServer无法处理该请求时交由DelegationTarget进行处理

## 接口
```go
// vendor/k8s.io/apiserver/pkg/server/genericapiserver.go
type DelegationTarget interface {
	// UnprotectedHandler returns a handler that is NOT protected by a normal chain
	UnprotectedHandler() http.Handler

	// PostStartHooks returns the post-start hooks that need to be combined
	PostStartHooks() map[string]postStartHookEntry

	// PreShutdownHooks returns the pre-stop hooks that need to be combined
	PreShutdownHooks() map[string]preShutdownHookEntry

	// HealthzChecks returns the healthz checks that need to be combined
	HealthzChecks() []healthz.HealthChecker

	// ListedPaths returns the paths for supporting an index
	ListedPaths() []string

	// NextDelegate returns the next delegationTarget in the chain of delegations
	NextDelegate() DelegationTarget

	// PrepareRun does post API installation setup steps. It calls recursively the same function of the delegates.
	PrepareRun() preparedGenericAPIServer

	// MuxAndDiscoveryCompleteSignals exposes registered signals that indicate if all known HTTP paths have been installed.
	MuxAndDiscoveryCompleteSignals() map[string]<-chan struct{}
}
```
1. UnprotectedHandler：获取该DelegationTarget未经Filter链封装过的原始handler
2. PostStartHooks：获取该DelegationTarget中所有的PostStartHooks
3. PreShutdownHooks：获取该DelegationTarget中所有的PreShutdownHooks
4. HealthzChecks：获取该DelegationTarget中所有的HealthzChecks
5. ListedPaths：获取该DelegationTarget中所有的path
6. PrepareRun：预运行该DelegationTarget
7. MuxAndDiscoveryCompleteSignals：返回该DelegationTarget中所有的HTTP路径的加载完成信号channel

## 各个服务传入的Delegation
```go
// cmd/kube-apiserver/app/server.go
apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
```
1. apiExtensionsServer接收的delegation是一个空delegation，该delegation直接返回404
2. kubeAPIServer接收的delegation是apiExtensionsServer.GenericAPIServer
3. aggregatorServer接收的delegation是kubeAPIServer.GenericAPIServer

## 服务之间的delegation关系
Aggregator -> KubeAPIServer -> APIExtensions

## GenericAPIServer中的Delegation
GenericAPIServer将会调用delegation的Unprotected()方法来获取其中的未经Filter封装的Handler，并将其作为自己的NotfoundHandler