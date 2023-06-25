# RESTMapper
RESTMapper保存了从GVR与GVK互相之间的映射关系

## 接口
```go
// vendor/k8s.io/apimachinery/pkg/api/meta/interfaces.go
type RESTMapper interface {
	// KindFor takes a partial resource and returns the single match.  Returns an error if there are multiple matches
	KindFor(resource schema.GroupVersionResource) (schema.GroupVersionKind, error)

	// KindsFor takes a partial resource and returns the list of potential kinds in priority order
	KindsFor(resource schema.GroupVersionResource) ([]schema.GroupVersionKind, error)

	// ResourceFor takes a partial resource and returns the single match.  Returns an error if there are multiple matches
	ResourceFor(input schema.GroupVersionResource) (schema.GroupVersionResource, error)

	// ResourcesFor takes a partial resource and returns the list of potential resource in priority order
	ResourcesFor(input schema.GroupVersionResource) ([]schema.GroupVersionResource, error)

	// RESTMapping identifies a preferred resource mapping for the provided group kind.
	RESTMapping(gk schema.GroupKind, versions ...string) (*RESTMapping, error)
	// RESTMappings returns all resource mappings for the provided group kind if no
	// version search is provided. Otherwise identifies a preferred resource mapping for
	// the provided version(s).
	RESTMappings(gk schema.GroupKind, versions ...string) ([]*RESTMapping, error)

	ResourceSingularizer(resource string) (singular string, err error)
}
```
1. KindFor：接收部分GVR(可能残缺)，返回对应的GVK，如果有多匹配，返回error
2. KindsFor：接收部分GVR(可能残缺)，返回对应的所有可能的GVK
3. ResourceFor：接收部分GVK(可能残缺)，返回对应的GVR，如果有多匹配，返回error
4. ResourcesFor：接收部分GVK(可能残缺)，返回对应的所有可能的GVR
5. RESTMapping：接收GK和多个V，返回首选的RESTMapping
6. RESTMappings：接收GK和多个V，返回所有的RESTMapping

## 实现类
1. DefaultRESTMapper：RESTMapper默认实现，由多个map结构组成
2. DeferredDiscoveryRESTMapper：对Discovery客户端进行封装，实现RESTMapper接口
