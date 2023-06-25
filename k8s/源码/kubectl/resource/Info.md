# Info
Info用于保存服务器返回的响应，操作Info的函数一般会作为Visitor的参数进行使用

## 结构体
```go
type Info struct {
	// Client will only be present if this builder was not local
	Client RESTClient
	// Mapping will only be present if this builder was not local
	Mapping *meta.RESTMapping

	// Namespace will be set if the object is namespaced and has a specified value.
	Namespace string
	Name      string

	// Optional, Source is the filename or URL to template file (.json or .yaml),
	// or stdin to use to handle the resource
	Source string
	// Optional, this is the most recent value returned by the server if available. It will
	// typically be in unstructured or internal forms, depending on how the Builder was
	// defined. If retrieved from the server, the Builder expects the mapping client to
	// decide the final form. Use the AsVersioned, AsUnstructured, and AsInternal helpers
	// to alter the object versions.
	Object runtime.Object
	// Optional, this is the most recent resource version the server knows about for
	// this type of resource. It may not match the resource version of the object,
	// but if set it should be equal to or newer than the resource version of the
	// object (however the server defines resource version).
	ResourceVersion string
}
```
1. Client：使用的客户端
2. Mapping：
3. Source：来源文件
4. Object：从服务器获取的响应，被反序列化成runtime.Object的实现类

## 方法
```go
// Info也实现了Visitor，直接执行传入的fn
func (i *Info) Visit(fn VisitorFunc) error {
	return fn(i, nil)
}

// RESTMapping
```