# APIService
APIService资源(`apiregistration.k8s.io/v1/APIService`)是k8s在AggregatorServer中提供一个扩展机制  
当AggregatorServer收到一个请求后，将会根据uri中的GV找到，并APIService资源的service字段，来决定将路由交由本地Server处理还是某个Service进行处理  
CRD被添加时也会被转化为APIService资源持久化

## 相关的Controller
1. crdRegsitrationController：Watch CRD资源对象并转换成APIService资源对象
2. autoRegisterController：调用回环客户端将APIService持久化到ETCD，并根据需要持续将传入的APIService进行同步
3. APIServiceRegistrationController：Watch APIService资源对象，将资源对象通知给AggregatorServer，AggregatorServer添加路由
4. kube-apiserver-autoregistration：一个PostStartHook，启动上述组件

## 分类
APIService分为两类：
1. 本地Server：APIService的Service字段为nil时，默认交由本地Server进行处理
2. 其他Server：APIService的Service字段不为nil时，交由指定的Service进行处理

## 来源
1. 启动时遍历k8s内置API所有的GV转化为APIService
2. 用户添加的CRD的GV转化为APIService
3. 通过RESTful接口直接添加APIService

## 结构体
```go
// vendor/k8s.io/kube-aggregator/pkg/apis/apiregistration/v1/types.go
type APIService struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Spec contains information for locating and communicating with a server
	Spec APIServiceSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
	// Status contains derived information about an API server
	Status APIServiceStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}

type APIServiceSpec struct {
	// 如果该字段为nil，则默认请求本地服务
	Service *ServiceReference `json:"service,omitempty" protobuf:"bytes,1,opt,name=service"`
	// Group is the API group name this server hosts
	Group string `json:"group,omitempty" protobuf:"bytes,2,opt,name=group"`
	// Version is the API version this server hosts.  For example, "v1"
	Version string `json:"version,omitempty" protobuf:"bytes,3,opt,name=version"`

	InsecureSkipTLSVerify bool `json:"insecureSkipTLSVerify,omitempty" protobuf:"varint,4,opt,name=insecureSkipTLSVerify"`
	CABundle []byte `json:"caBundle,omitempty" protobuf:"bytes,5,opt,name=caBundle"`

	// 组优先级
	GroupPriorityMinimum int32 `json:"groupPriorityMinimum" protobuf:"varint,7,opt,name=groupPriorityMinimum"`

	// 版本优先级
	VersionPriority int32 `json:"versionPriority" protobuf:"varint,8,opt,name=versionPriority"`

}

type ServiceReference struct {
	// Namespace is the namespace of the service
	// Service的命名空间
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,1,opt,name=namespace"`
	// Name is the name of the service
	// Service的名字
	Name string `json:"name,omitempty" protobuf:"bytes,2,opt,name=name"`
	// If specified, the port on the service that hosting webhook.
	// Default to 443 for backward compatibility.
	// `port` should be a valid port number (1-65535, inclusive).
	// +optional
	Port *int32 `json:"port,omitempty" protobuf:"varint,3,opt,name=port"`
}
```