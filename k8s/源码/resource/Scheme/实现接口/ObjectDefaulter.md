# ObjectDefaulter
传入一个runtime.Object，给其赋默认值

## 接口
```go
type ObjectDefaulter interface {
	Default(in Object)
}
```

## 实现
```go
// k8s.io/apimachinery/pkg/runtime/scheme.go
func (s *Scheme) Default(src Object) {
    // 从goType中获取对应的默认值函数，如果有，则调用
	if fn, ok := s.defaulterFuncs[reflect.TypeOf(src)]; ok {
		fn(src)
	}
}
```