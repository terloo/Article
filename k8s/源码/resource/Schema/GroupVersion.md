# GroupVersion
实现了runtime.GroupVersioner接口

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
type GroupVersion struct {
	Group   string
	Version string
}
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
func (gv GroupVersion) Empty() bool {
	return len(gv.Group) == 0 && len(gv.Version) == 0
}

func (gv GroupVersion) String() string {
	if len(gv.Group) > 0 {
		return gv.Group + "/" + gv.Version
	}
	return gv.Version
}

func (gv GroupVersion) WithKind(kind string) GroupVersionKind {
	return GroupVersionKind{Group: gv.Group, Version: gv.Version, Kind: kind}
}

func (gv GroupVersion) WithResource(resource string) GroupVersionResource {
	return GroupVersionResource{Group: gv.Group, Version: gv.Version, Resource: resource}
}

// 实现runtime.GroupVersioner
func (gv GroupVersion) Identifier() string {
	return gv.String()
}

// 实现runtime.GroupVersioner
func (gv GroupVersion) KindForGroupVersionKinds(kinds []GroupVersionKind) (target GroupVersionKind, ok bool) {
	// 遍历第一次，如果有kinds的GV与该GV完全匹配，直接返回
	for _, gvk := range kinds {
		if gvk.Group == gv.Group && gvk.Version == gv.Version {
			return gvk, true
		}
	}

	// 遍历第二次，如果Group能匹配，也算匹配成功
	for _, gvk := range kinds {
		if gvk.Group == gv.Group {
			return gv.WithKind(gvk.Kind), true
		}
	}
	return GroupVersionKind{}, false
}
```