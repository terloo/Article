# ObjectTyper
传入一个runtime.Object，返回其所有可能的GVK；判断某个GVK是否已经注册到该scheme

## 接口
```go
type ObjectTyper interface {
	ObjectKinds(Object) ([]schema.GroupVersionKind, bool, error)
	Recognizes(gvk schema.GroupVersionKind) bool
}
```
1. ObjectKinds：返回true如果是无版本资源，error如果资源未注册或者传入的不是指针类型
2. Recognizes：返回资源是否已注册

## 实现
```go
// k8s.io/apimachinery/pkg/runtime/scheme.go
func (s *Scheme) ObjectKinds(obj Object) ([]schema.GroupVersionKind, bool, error) {
	// Unstructured objects are always considered to have their declared GVK
	if _, ok := obj.(Unstructured); ok {
        // 如果runtime.Object是非结构化，则直接获取其GVK并返回
		// we require that the GVK be populated in order to recognize the object
		gvk := obj.GetObjectKind().GroupVersionKind()
		if len(gvk.Kind) == 0 {
			return nil, false, NewMissingKindErr("unstructured object has no kind")
		}
		if len(gvk.Version) == 0 {
			return nil, false, NewMissingVersionErr("unstructured object has no version")
		}
		return []schema.GroupVersionKind{gvk}, false, nil
	}

    // 获取指针对应的对象的goType
	v, err := conversion.EnforcePtr(obj)
	if err != nil {
		return nil, false, err
	}
	t := v.Type()

	gvks, ok := s.typeToGVK[t]
	if !ok {
		return nil, false, NewNotRegisteredErrForType(s.schemeName, t)
	}
	_, unversionedType := s.unversionedTypes[t]

	return gvks, unversionedType, nil
}

func (s *Scheme) Recognizes(gvk schema.GroupVersionKind) bool {
    // 由于gvkToType中同时保存了无版本和普通资源对象，所以只需要判断gvkToType中有无该gvk
	_, exists := s.gvkToType[gvk]
	return exists
}
```