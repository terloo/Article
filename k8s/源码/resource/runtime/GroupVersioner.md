# GroupVersioner
从一组GVK的集合中(根据结构体自身属性)挑选出最合适的一个GVK并返回，返回的GVK不一定在输出的GVK列表中

## 接口
```go
// k8s.io/apimachinery/pkg/runtime/interfaces.go
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

## 实现类
1. GroupVersion：GV匹配即可返回，GV不匹配可以只匹配G然后赋值K
2. GroupVersions：循环调用GroupVersion的KindForGroupVersionKinds，然后提取出最佳匹配
