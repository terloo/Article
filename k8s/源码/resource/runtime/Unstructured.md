# Unstructured
Unstructured接口组合了`runtime.Object`接口，用于接收无法判断类型的资源对象实例

## 接口
```go
type Unstructured interface {
	Object
	// NewEmptyInstance returns a new instance of the concrete type containing only kind/apiVersion and no other data.
	// This should be called instead of reflect.New() for unstructured types because the go type alone does not preserve kind/apiVersion info.
	NewEmptyInstance() Unstructured
	// UnstructuredContent returns a non-nil map with this object's contents. Values may be
	// []interface{}, map[string]interface{}, or any primitive type. Contents are typically serialized to
	// and from JSON. SetUnstructuredContent should be used to mutate the contents.
	UnstructuredContent() map[string]interface{}
	// SetUnstructuredContent updates the object content to match the provided map.
	SetUnstructuredContent(map[string]interface{})
	// IsList returns true if this type is a list or matches the list convention - has an array called "items".
	IsList() bool
	// EachListItem should pass a single item out of the list as an Object to the provided function. Any
	// error should terminate the iteration. If IsList() returns false, this method should return an error
	// instead of calling the provided function.
	EachListItem(func(Object) error) error
}
```
1. Object：组合的`runtime.Object`
2. NewEmptyInstance：仅返回一个包含kind/apiVersion的Unstructured对象
3. UnstructuredContent：返回该Unstructured对象的map表现形式
4. IsList：返回对象是否含有一个列表字段items
5. EachListItem：对List中的每一个对象进行一次函数调用

## 实现类

### Unstructured
用于反序列化普通对象
```go
type Unstructured struct {
	// Object is a JSON compatible map with string, float, int, bool, []interface{}, or
	// map[string]interface{}
	// children.
	Object map[string]interface{}
}
```

### UnstructuredList
用于反序列化列表对象
```go
type UnstructuredList struct {
	Object map[string]interface{}

	// Items is a list of unstructured objects.
	Items []Unstructured `json:"items"`
}
```