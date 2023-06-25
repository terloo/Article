# runtime.Object
runtime.Object是k8s所有资源的基础接口，位于`vendor/k8s.io/apimachinery/pkg/runtime/interface.go`  
由于`schema.ObjectKind`接口和`metav1.Object`接口分别用于资源对象类型和元数据的表现，所有还需要一个接口用于接收整个对象

## runtime.Object结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go
type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}
```
1. GetObjectKind：获取资源对象的类型，以`schema.ObjectKind`接口的形式返回
2. DeepCopyObject：深拷贝对象，返回自身