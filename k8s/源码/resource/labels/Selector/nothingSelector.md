# nothingSelector
nothingSelector不会匹配任何的标签组

## 结构体
```go
// k8s.io/apimachinery/pkg/labels/selector.go
type nothingSelector struct{}
```

## 方法
```go
// k8s.io/apimachinery/pkg/labels/selector.go
func (n nothingSelector) Matches(_ Labels) bool              { return false }
func (n nothingSelector) Empty() bool                        { return false }
func (n nothingSelector) String() string                     { return "" }
func (n nothingSelector) Add(_ ...Requirement) Selector      { return n }
func (n nothingSelector) Requirements() (Requirements, bool) { return nil, false }
func (n nothingSelector) DeepCopySelector() Selector         { return n }
func (n nothingSelector) RequiresExactMatch(label string) (value string, found bool) {
	return "", false
}
```

## Nothing
返回一个不会匹配任何标签组的Selector
```go
// k8s.io/apimachinery/pkg/labels/selector.go
func Nothing() Selector {
	return nothingSelector{}
}
```