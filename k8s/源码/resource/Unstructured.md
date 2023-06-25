# Unstructured数据

## 非结构化数据
非结构化数据(Unstructured Data)是无法预知数据类型或属性名称不确定的数据类型，其无法通过构建预定的struct来序列化或反序列化数据

## 非结构化数据处理
```go
// 非结构化数据的接口
// vendor/k8s.io/apimachinery/pkg/runtime/interfaces.go
type Unstructured interface {
    // 指runtime.Object
	Object
	// 返回只包含了GVK信息的新实例
	NewEmptyInstance() Unstructured
    // 返回非结构化数据信息
	UnstructuredContent() map[string]interface{}
	// 设置新的非结构化数据信息
	SetUnstructuredContent(map[string]interface{})
	// 是否为对象集合
	IsList() bool
	// 对每个对象进行操作，如果IsList返回false，会返回error
	EachListItem(func(Object) error) error
}

// vendor/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go
// 非结构化数据接口对应的实现类
```