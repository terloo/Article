# Accessor
由于k8s资源对象和资源对象List实现的元数据接口不一致，所以无法通过一个统一的接口来访问元数据。k8s提供了Accessor系列函数来访问资源对象的元数据。

## CommonAccessor
通过接口断言，返回Common接口
```go
func CommonAccessor(obj interface{}) (metav1.Common, error) {
	switch t := obj.(type) {
	case List:
		return t, nil
	case ListMetaAccessor:
		if m := t.GetListMeta(); m != nil {
			return m, nil
		}
		return nil, errNotCommon
	case metav1.ListMetaAccessor:
		if m := t.GetListMeta(); m != nil {
			return m, nil
		}
		return nil, errNotCommon
	case metav1.Object:
		return t, nil
	case metav1.ObjectMetaAccessor:
		if m := t.GetObjectMeta(); m != nil {
			return m, nil
		}
		return nil, errNotCommon
	default:
		return nil, errNotCommon
	}
}
```

## ListAccessor
如果知道一个资源对象是List，可以返回List接口
```go
func ListAccessor(obj interface{}) (List, error) {
	switch t := obj.(type) {
	case List:
		return t, nil
	case ListMetaAccessor:
		if m := t.GetListMeta(); m != nil {
			return m, nil
		}
		return nil, errNotList
	case metav1.ListMetaAccessor:
		if m := t.GetListMeta(); m != nil {
			return m, nil
		}
		return nil, errNotList
	default:
		return nil, errNotList
	}
}
```

## Accessor
如果知道一个资源对象是普通资源对象，可以返回metav1.Object
```go
// k8s.io/apimachinery/pkg/api/meta/meta.go
func Accessor(obj interface{}) (metav1.Object, error) {
	switch t := obj.(type) {
    case metav1.Object:
    // 如果对象实现了metav1.Object，则直接返回。大多数标准对象满足此条件
		return t, nil
	case metav1.ObjectMetaAccessor:
    // 或者对象提供了获取metav1.Object形式的方法，调用该方法获取后返回。没有组合ObjectMeta却是现实了ObjectMetaAccessor
		if m := t.GetObjectMeta(); m != nil {
			return m, nil
		}
		return nil, errNotObject
	default:
		return nil, errNotObject
	}
}
```
