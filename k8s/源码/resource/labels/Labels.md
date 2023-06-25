# Labels
Labels定义了k8s中标签对象的能力

## 接口
```go
// k8s.io/apimachinery/pkg/labels/labels.go
type Labels interface {
	// Has returns whether the provided label exists.
	Has(label string) (exists bool)

	// Get returns the value for the provided label.
	Get(label string) (value string)
}
```

## 实现类
Set实现了Labels，用于保存多组标签
```go
// k8s.io/apimachinery/pkg/labels/labels.go
// Set is a map of label:value. It implements Labels.
type Set map[string]string

// String returns all labels listed as a human readable string.
// Conveniently, exactly the format that ParseSelector takes.
func (ls Set) String() string {
	selector := make([]string, 0, len(ls))
	for key, value := range ls {
		selector = append(selector, key+"="+value)
	}
	// Sort for determinism.
	sort.StringSlice(selector).Sort()
	return strings.Join(selector, ",")
}

// Has returns whether the provided label exists in the map.
func (ls Set) Has(label string) bool {
	_, exists := ls[label]
	return exists
}

// Get returns the value in the map for the provided label.
func (ls Set) Get(label string) string {
	return ls[label]
}
```

## 相关操作函数
```go
// k8s.io/apimachinery/pkg/labels/labels.go
// 合并两组标签，后一组标签优先级更高
func Merge(labels1, labels2 Set) Set {
	mergedMap := Set{}

	for k, v := range labels1 {
		mergedMap[k] = v
	}
	for k, v := range labels2 {
		mergedMap[k] = v
	}
	return mergedMap
}

// 判断两组标签是否相等
func Equals(labels1, labels2 Set) bool {
	if len(labels1) != len(labels2) {
		return false
	}

	for k, v := range labels1 {
		value, ok := labels2[k]
		if !ok {
			return false
		}
		if value != v {
			return false
		}
	}
	return true
}
```