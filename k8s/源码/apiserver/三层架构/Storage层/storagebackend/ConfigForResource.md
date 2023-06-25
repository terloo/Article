# ConfigForResource
每个GR都会使用一份ConfigForResource来创建属于该GR的store实例

## 结构体
```go
// k8s.io/apiserver/pkg/storage/storagebackend/config.go
type ConfigForResource struct {
	// Config is the resource-independent configuration
	Config

	// GroupResource is the relevant one
	GroupResource schema.GroupResource
}
```
1. Config：Config实例，非指针类型，由基础Config拷贝得来
2. GroupResource：该ConfigForResource对应的GR