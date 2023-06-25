# internalSelector
internalSelector是Selector的实现类，拥有多个Requirement

## 结构体
```go
// k8s.io/apimachinery/pkg/labels/selector.go
type internalSelector []Requirement
```

## 方法
```go
// k8s.io/apimachinery/pkg/labels/selector.go

// 核心方法，实现了Selector.Matches
func (s internalSelector) Matches(l Labels) bool {
    // 遍历拥有的所有Requirement，取与逻辑
	for ix := range s {
		if matches := s[ix].Matches(l); !matches {
			return false
		}
	}
	return true
}
```

## 字符串形式
```go
// k8s.io/apimachinery/pkg/labels/selector.go
func (s internalSelector) String() string {
	var reqs []string
	for ix := range s {
		reqs = append(reqs, s[ix].String())
	}
	return strings.Join(reqs, ",")
}
```

## Everything
新建一个能匹配所有标签组的Selector
```go
// k8s.io/apimachinery/pkg/labels/selector.go
func Everything() Selector {
	return internalSelector{}
}
```
