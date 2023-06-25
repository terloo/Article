# GroupVersioner
GroupVersioner定义了对多个GVK进行筛选的功能，在无法确定最佳的GVK时，调用此接口可以挑选出最佳的GVK

## 接口
```go
// vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go
type GroupVersioner interface {
	// KindForGroupVersionKinds returns a desired target group version kind for the given input, or returns ok false if no
	// target is known. In general, if the return target is not in the input list, the caller is expected to invoke
	// Scheme.New(target) and then perform a conversion between the current Go type and the destination Go type.
	// Sophisticated implementations may use additional information about the input kinds to pick a destination kind.
	KindForGroupVersionKinds(kinds []schema.GroupVersionKind) (target schema.GroupVersionKind, ok bool)
	// Identifier returns string representation of the object.
	// Identifiers of two different encoders should be equal only if for every input
	// kinds they return the same result.
	Identifier() string
}
```
1. KindForGroupVersionKinds：用于寻找多个GVK中与该GV的最佳匹配，返回的GVK可能不在输入的GVK列表中