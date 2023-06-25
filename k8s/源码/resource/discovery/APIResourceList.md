# APIResourceList

## Discovery路径
1. `/apis/<group_name>/<version_name>`
2. `/api/v1`

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/apis/meta/v1/types.go
type APIResourceList struct {
	TypeMeta `json:",inline"`
	// groupVersion is the group and version this APIResourceList is for.
	GroupVersion string `json:"groupVersion" protobuf:"bytes,1,opt,name=groupVersion"`
	// resources contains the name of the resources and if they are namespaced.
	APIResources []APIResource `json:"resources" protobuf:"bytes,2,rep,name=resources"`
}

// 保存GroupVersion中所有资源的Resource名、Kind名、可进行的操作、是否为ns级资源
type APIResource struct {
	// 资源名称，通常为复数形式，子资源名称为 父资源/子资源
	Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
	// 资源的名称单数形式，必须小写，默认由Kind的小写形式
	SingularName string `json:"singularName" protobuf:"bytes,6,opt,name=singularName"`
	// 是否是命名空间级别
	Namespaced bool `json:"namespaced" protobuf:"varint,2,opt,name=namespaced"`
	// group is the preferred group of the resource.  Empty implies the group of the containing resource list.
	// For subresources, this may have a different value, for example: Scale".
	Group string `json:"group,omitempty" protobuf:"bytes,8,opt,name=group"`
	// version is the preferred version of the resource.  Empty implies the version of the containing resource list
	// For subresources, this may have a different value, for example: v1 (while inside a v1beta1 version of the core resource's group)".
	Version string `json:"version,omitempty" protobuf:"bytes,9,opt,name=version"`
	// kind is the kind for the resource (e.g. 'Foo' is the kind for a resource 'foo')
	Kind string `json:"kind" protobuf:"bytes,3,opt,name=kind"`
	// 能对资源进行的操作
	Verbs Verbs `json:"verbs" protobuf:"bytes,4,opt,name=verbs"`
	// 资源的简称
	ShortNames []string `json:"shortNames,omitempty" protobuf:"bytes,5,rep,name=shortNames"`
	// 资源所属分类，例如{"all"}，便于命令行操作
	Categories []string `json:"categories,omitempty" protobuf:"bytes,7,rep,name=categories"`
	// The hash value of the storage version, the version this resource is
	// converted to when written to the data store. Value must be treated
	// as opaque by clients. Only equality comparison on the value is valid.
	// This is an alpha feature and may change or be removed in the future.
	// The field is populated by the apiserver only if the
	// StorageVersionHash feature gate is enabled.
	// This field will remain optional even if it graduates.
	// +optional
	StorageVersionHash string `json:"storageVersionHash,omitempty" protobuf:"bytes,10,opt,name=storageVersionHash"`
}
```
