# GroupVersions

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
type GroupVersions []GroupVersion
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go
func (gvs GroupVersions) Identifier() string {
	groupVersions := make([]string, 0, len(gvs))
	for i := range gvs {
		groupVersions = append(groupVersions, gvs[i].String())
	}
	return fmt.Sprintf("[%s]", strings.Join(groupVersions, ","))
}

func (gvs GroupVersions) KindForGroupVersionKinds(kinds []GroupVersionKind) (GroupVersionKind, bool) {
	var targets []GroupVersionKind
	for _, gv := range gvs {
        // 遍历查找GroupVersions中能匹配到的所有kinds
		target, ok := gv.KindForGroupVersionKinds(kinds)
		if !ok {
			continue
		}
		targets = append(targets, target)
	}
	if len(targets) == 1 {
		return targets[0], true
	}
    // 如果能匹配到多个，查找第一个精准匹配
	if len(targets) > 1 {
		return bestMatch(kinds, targets), true
	}
	return GroupVersionKind{}, false
}

// bestMatch tries to pick best matching GroupVersionKind and falls back to the first
// found if no exact match exists.
// 查找第一个精准匹配，如果查找不到，则返回第一个匹配
func bestMatch(kinds []GroupVersionKind, targets []GroupVersionKind) GroupVersionKind {
	for _, gvk := range targets {
		for _, k := range kinds {
			if k == gvk {
				return k
			}
		}
	}
	return targets[0]
}
```