# RESTMapping
RESTMapping保存一个资源对象类型在进行REST操作时所需要的信息

## 结构体
```go
type RESTMapping struct {
	// Resource is the GroupVersionResource (location) for this endpoint
	Resource schema.GroupVersionResource

	// GroupVersionKind is the GroupVersionKind (data format) to submit to this endpoint
	GroupVersionKind schema.GroupVersionKind

	// Scope contains the information needed to deal with REST Resources that are in a resource hierarchy
	Scope RESTScope
}
```
1. Resource：资源的GVR
2. GroupVersionKind：资源的GVK
3. Scope：资源的作用范围

## 资源的作用范围
```go
const (
    // 命名空间级别
	RESTScopeNameNamespace RESTScopeName = "namespace"
    // 集群级别
	RESTScopeNameRoot      RESTScopeName = "root"
)

// RESTScope contains the information needed to deal with REST resources that are in a resource hierarchy
type RESTScope interface {
	// Name of the scope
	Name() RESTScopeName
}
```
