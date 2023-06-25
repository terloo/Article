# ListMeta
ListMeta是用于包装k8s资源集合的结构体，所有资源都拥有对应的XXXList结构体，该结构体必须组合ListMeta结构体

## 结构体
```go
// k8s.io/apimachinery/pkg/apis/meta/v1/types.go
// ListMeta describes metadata that synthetic resources must have, including lists and
// various status objects. A resource may have only one of {ObjectMeta, ListMeta}.
type ListMeta struct {
	// selfLink is a URL representing this object.
	// Populated by the system.
	// Read-only.
	//
	// DEPRECATED
	// Kubernetes will stop propagating this field in 1.20 release and the field is planned
	// to be removed in 1.21 release.
	// +optional
	SelfLink string `json:"selfLink,omitempty" protobuf:"bytes,1,opt,name=selfLink"`

	// String that identifies the server's internal version of this object that
	// can be used by clients to determine when objects have changed.
	// Value must be treated as opaque by clients and passed unmodified back to the server.
	// Populated by the system.
	// Read-only.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#concurrency-control-and-consistency
	// +optional
	ResourceVersion string `json:"resourceVersion,omitempty" protobuf:"bytes,2,opt,name=resourceVersion"`

	// continue may be set if the user set a limit on the number of items returned, and indicates that
	// the server has more data available. The value is opaque and may be used to issue another request
	// to the endpoint that served this list to retrieve the next set of available objects. Continuing a
	// consistent list may not be possible if the server configuration has changed or more than a few
	// minutes have passed. The resourceVersion field returned when using this continue value will be
	// identical to the value in the first response, unless you have received this token from an error
	// message.
	Continue string `json:"continue,omitempty" protobuf:"bytes,3,opt,name=continue"`

	// remainingItemCount is the number of subsequent items in the list which are not included in this
	// list response. If the list request contained label or field selectors, then the number of
	// remaining items is unknown and the field will be left unset and omitted during serialization.
	// If the list is complete (either because it is not chunking or because this is the last chunk),
	// then there are no more remaining items and this field will be left unset and omitted during
	// serialization.
	// Servers older than v1.15 do not set this field.
	// The intended use of the remainingItemCount is *estimating* the size of a collection. Clients
	// should not rely on the remainingItemCount to be set or to be exact.
	// +optional
	RemainingItemCount *int64 `json:"remainingItemCount,omitempty" protobuf:"bytes,4,opt,name=remainingItemCount"`
}
```

## 接口
```go
// k8s.io/apimachinery/pkg/apis/meta/v1/meta.go
// ListMetaAccessor retrieves the list interface from an object
type ListMetaAccessor interface {
	GetListMeta() ListInterface
}

// ListInterface lets you work with list metadata from any of the versioned or
// internal API objects. Attempting to set or retrieve a field on an object that does
// not support that field will be a no-op and return a default value.
type ListInterface interface {
	GetResourceVersion() string
	SetResourceVersion(version string)
	GetSelfLink() string
	SetSelfLink(selfLink string)
	GetContinue() string
	SetContinue(c string)
	GetRemainingItemCount() *int64
	SetRemainingItemCount(c *int64)
}
```

## 方法
```go
// k8s.io/apimachinery/pkg/apis/meta/v1/meta.go
func (obj *ListMeta) GetListMeta() ListInterface { return obj }

func (meta *ListMeta) GetResourceVersion() string        { return meta.ResourceVersion }
func (meta *ListMeta) SetResourceVersion(version string) { meta.ResourceVersion = version }
func (meta *ListMeta) GetSelfLink() string               { return meta.SelfLink }
func (meta *ListMeta) SetSelfLink(selfLink string)       { meta.SelfLink = selfLink }
func (meta *ListMeta) GetContinue() string               { return meta.Continue }
func (meta *ListMeta) SetContinue(c string)              { meta.Continue = c }
func (meta *ListMeta) GetRemainingItemCount() *int64     { return meta.RemainingItemCount }
func (meta *ListMeta) SetRemainingItemCount(c *int64)    { meta.RemainingItemCount = c }
```