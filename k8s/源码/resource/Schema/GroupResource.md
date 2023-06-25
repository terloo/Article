# GroupResource

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
type GroupResource struct {
	Group    string
	Resource string
}
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
// 接收一个Version，返回GVR
func (gr GroupResource) WithVersion(version string) GroupVersionResource {
	return GroupVersionResource{Group: gr.Group, Version: version, Resource: gr.Resource}
}

func (gr GroupResource) Empty() bool {
	return len(gr.Group) == 0 && len(gr.Resource) == 0
}

func (gr GroupResource) String() string {
	if len(gr.Group) == 0 {
		return gr.Resource
	}
	return gr.Resource + "." + gr.Group
}
```