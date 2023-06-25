# GroupVersionResource

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
type GroupVersionResource struct {
	Group    string
	Version  string
	Resource string
}
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
func (gvr GroupVersionResource) Empty() bool {
	return len(gvr.Group) == 0 && len(gvr.Version) == 0 && len(gvr.Resource) == 0
}

func (gvr GroupVersionResource) GroupResource() GroupResource {
	return GroupResource{Group: gvr.Group, Resource: gvr.Resource}
}

func (gvr GroupVersionResource) GroupVersion() GroupVersion {
	return GroupVersion{Group: gvr.Group, Version: gvr.Version}
}

func (gvr GroupVersionResource) String() string {
	return strings.Join([]string{gvr.Group, "/", gvr.Version, ", Resource=", gvr.Resource}, "")
}
```