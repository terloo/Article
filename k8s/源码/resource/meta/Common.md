# Common
Common是k8s中所有资源对象及资源List对象都通过组合`ObjectMeta`或者`ListMeta`来实现该接口

## 接口
```go
// k8s.io/apimachinery/pkg/apis/meta/v1/meta.go
type Common interface {
	GetResourceVersion() string
	SetResourceVersion(version string)
	GetSelfLink() string
	SetSelfLink(selfLink string)
}
```