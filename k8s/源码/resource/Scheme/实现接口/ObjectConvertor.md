# ObjectConvertor
资源的版本转化相关方法

## 接口
```go
type ObjectConvertor interface {
	Convert(in, out, context interface{}) error
	ConvertToVersion(in Object, gv GroupVersioner) (out Object, err error)
	ConvertFieldLabel(gvk schema.GroupVersionKind, label, value string) (string, string, error)
}
```
1. Convert：尝试将in的类型转化为out类型
2. ConvertToVersion：ObjectVersioner接口
3. ConvertFieldLabel：将传入的label和value