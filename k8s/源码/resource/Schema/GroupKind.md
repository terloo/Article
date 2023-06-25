# GroupKind

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
type GroupKind struct {
	Group string
	Kind  string
}
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
func (gk GroupKind) Empty() bool {
	return len(gk.Group) == 0 && len(gk.Kind) == 0
}

func (gk GroupKind) WithVersion(version string) GroupVersionKind {
	return GroupVersionKind{Group: gk.Group, Version: version, Kind: gk.Kind}
}

func (gk GroupKind) String() string {
	if len(gk.Group) == 0 {
		return gk.Kind
	}
	return gk.Kind + "." + gk.Group
}
```