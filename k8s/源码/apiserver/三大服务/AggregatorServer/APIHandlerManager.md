# APIHandlerManager
APIHandlerManager是Aggregator接口，定义了向Server中添加APIService和移除APIService的方法

## 接口
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/apiserver_controller.go
type APIHandlerManager interface {
	AddAPIService(apiService *v1.APIService) error
	RemoveAPIService(apiServiceName string)
}
```
1. AddAPIService：添加一个APIService资源对象
2. RemoveAPIService：移除一个APIService资源对象
