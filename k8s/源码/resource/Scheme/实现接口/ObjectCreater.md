# ObjectCreater
传入一个GVK，返回其对应的runtime.Object实例

## 接口
```go
type ObjectCreater interface {
	New(kind schema.GroupVersionKind) (out Object, err error)
}
```

## 实现
```go
// k8s.io/apimachinery/pkg/runtime/scheme.go
func (s *Scheme) New(kind schema.GroupVersionKind) (Object, error) {
    // 尝试从gvkToType中获取对应的goType
	if t, exists := s.gvkToType[kind]; exists {
		return reflect.New(t).Interface().(Object), nil
	}

    // 尝试使用Kind从unversionedKinds中获取对应的goType
	if t, exists := s.unversionedKinds[kind.Kind]; exists {
		return reflect.New(t).Interface().(Object), nil
	}
	return nil, NewNotRegisteredErrForKind(s.schemeName, kind)
}
```