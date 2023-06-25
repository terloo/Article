# ObjectVersioner
传入一个runtime.Object，尝试将其转化为相同Kind的另一个GV

## 接口
```go
type ObjectVersioner interface {
	ConvertToVersion(in Object, gv GroupVersioner) (out Object, err error)
}
```

## 实现
```go
// k8s.io/apimachinery/pkg/runtime/scheme.go
func (s *Scheme) ConvertToVersion(in Object, target GroupVersioner) (Object, error) {
    // 调用convertToVersion，是否拷贝设置为true
	return s.convertToVersion(true, in, target)
}

// ConvertToVersion的另一个方法，不是接口的实现
func (s *Scheme) UnsafeConvertToVersion(in Object, target GroupVersioner) (Object, error) {
    // 调用convertToVersion，是否拷贝设置为false
	return s.convertToVersion(false, in, target)
}

func (s *Scheme) convertToVersion(copy bool, in Object, target GroupVersioner) (Object, error) {
	var t reflect.Type

	if u, ok := in.(Unstructured); ok {
        // 将非结构化的数据给解析为结构化的数据
		typed, err := s.unstructuredToTyped(u)
		if err != nil {
			return nil, err
		}

        // 用结构化的数据替换掉原始传入的in
		in = typed
		// unstructuredToTyped returns an Object, which must be a pointer to a struct.
		t = reflect.TypeOf(in).Elem()

	} else {
		// determine the incoming kinds with as few allocations as possible.
		t = reflect.TypeOf(in)
		if t.Kind() != reflect.Ptr {
			return nil, fmt.Errorf("only pointer types may be converted: %v", t)
		}
		t = t.Elem()
		if t.Kind() != reflect.Struct {
			return nil, fmt.Errorf("only pointers to struct types may be converted: %v", t)
		}
	}

    // 获得真实的goType之后通过typeToGVK来获取该goType可能对应的所有GVK
	kinds, ok := s.typeToGVK[t]
	if !ok || len(kinds) == 0 {
		return nil, NewNotRegisteredErrForType(s.schemeName, t)
	}

    // 判断所有可能的GVK中的GV或者G是否与转化目标GV是否有相同，如果没有相同的，说明普通资源肯定无法完成转化
	gvk, ok := target.KindForGroupVersionKinds(kinds)
	if !ok {
        // try to see if this type is listed as unversioned (for legacy support)
		// TODO: when we move to server API versions, we should completely remove the unversioned concept
        // 在无法转化时，传入的Object是非版本化的，则直接转成非版本化即可
		if unversionedKind, ok := s.unversionedTypes[t]; ok {
			if gvk, ok := target.KindForGroupVersionKinds([]schema.GroupVersionKind{unversionedKind}); ok {
				return copyAndSetTargetKind(copy, in, gvk)
			}
			return copyAndSetTargetKind(copy, in, unversionedKind)
		}
		return nil, NewNotRegisteredErrForTarget(s.schemeName, t, target)
	}

	// target wants to use the existing type, set kind and return (no conversion necessary)
    // 如果GVK完全相同，说明不需要转化
	for _, kind := range kinds {
		if gvk == kind {
			return copyAndSetTargetKind(copy, in, gvk)
		}
	}

	// type is unversioned, no conversion necessary
	if unversionedKind, ok := s.unversionedTypes[t]; ok {
		if gvk, ok := target.KindForGroupVersionKinds([]schema.GroupVersionKind{unversionedKind}); ok {
			return copyAndSetTargetKind(copy, in, gvk)
		}
		return copyAndSetTargetKind(copy, in, unversionedKind)
	}

	out, err := s.New(gvk)
	if err != nil {
		return nil, err
	}

	if copy {
		in = in.DeepCopyObject()
	}

    // new一个out对象，然后转化
	meta := s.generateConvertMeta(in)
	meta.Context = target
	if err := s.converter.Convert(in, out, meta); err != nil {
		return nil, err
	}

	setTargetKind(out, gvk)
	return out, nil
}
```