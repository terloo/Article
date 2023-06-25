# SelectionPredicate
SelectionPredicate是apiServer中用于筛选从etcd中List的所有对象的结构体

## 结构体
```go
// k8s.io/apiserver/pkg/storage/selection_predicate.go
type SelectionPredicate struct {
	Label               labels.Selector
	Field               fields.Selector
	GetAttrs            AttrFunc
	IndexLabels         []string
	IndexFields         []string
	Limit               int64
	Continue            string
	AllowWatchBookmarks bool
}

type AttrFunc func(obj runtime.Object) (labels.Set, fields.Set, error)
```
1. Label：标签选择器
2. Field：字段选择器
3. GetAttrs：一个用于从对象中提取labels和fields的方法

## Match
接受一个对象，返回该对象能否被SelectionPredicate所匹配
```go
// k8s.io/apiserver/pkg/storage/selection_predicate.go
func (s *SelectionPredicate) Matches(obj runtime.Object) (bool, error) {
	if s.Empty() {
		return true, nil
	}
    // 先从对象中提取出对应的labels和fields
	labels, fields, err := s.GetAttrs(obj)
	if err != nil {
		return false, err
	}
    // 先判断labels能否匹配，如果能再判断fields能否匹配
	matched := s.Label.Matches(labels)
	if matched && s.Field != nil {
		matched = matched && s.Field.Matches(fields)
	}
	return matched, nil
}
```

## GetAttrs默认实现
```go
// k8s.io/apiserver/pkg/storage/selection_predicate.go
// 集群对象只提取metadata.name作为fields
func DefaultClusterScopedAttr(obj runtime.Object) (labels.Set, fields.Set, error) {
	metadata, err := meta.Accessor(obj)
	if err != nil {
		return nil, nil, err
	}
	fieldSet := fields.Set{
		"metadata.name": metadata.GetName(),
	}

	return labels.Set(metadata.GetLabels()), fieldSet, nil
}

// 命名空间级别对象还需提取metadata.namespace作为fields
func DefaultNamespaceScopedAttr(obj runtime.Object) (labels.Set, fields.Set, error) {
	metadata, err := meta.Accessor(obj)
	if err != nil {
		return nil, nil, err
	}
	fieldSet := fields.Set{
		"metadata.name":      metadata.GetName(),
		"metadata.namespace": metadata.GetNamespace(),
	}

	return labels.Set(metadata.GetLabels()), fieldSet, nil
}
```

## GetAttrs GK实现
以Pod实现的GetAttr为例
```go
// pkg/registry/core/pod/strategy.go

// 用于store中PredicateFunc字段
func MatchPod(label labels.Selector, field fields.Selector) storage.SelectionPredicate {
	return storage.SelectionPredicate{
		Label:       label,
		Field:       field,
		GetAttrs:    GetAttrs,
		IndexFields: []string{"spec.nodeName"},
	}
}

// GetAttrs实现
func GetAttrs(obj runtime.Object) (labels.Set, fields.Set, error) {
	pod, ok := obj.(*api.Pod)
	if !ok {
		return nil, nil, fmt.Errorf("not a pod")
	}
	return labels.Set(pod.ObjectMeta.Labels), ToSelectableFields(pod), nil
}

// Pod需要作为fields的属性很多
func ToSelectableFields(pod *api.Pod) fields.Set {
	// The purpose of allocation with a given number of elements is to reduce
	// amount of allocations needed to create the fields.Set. If you add any
	// field here or the number of object-meta related fields changes, this should
	// be adjusted.
	podSpecificFieldsSet := make(fields.Set, 9)
	podSpecificFieldsSet["spec.nodeName"] = pod.Spec.NodeName
	podSpecificFieldsSet["spec.restartPolicy"] = string(pod.Spec.RestartPolicy)
	podSpecificFieldsSet["spec.schedulerName"] = string(pod.Spec.SchedulerName)
	podSpecificFieldsSet["spec.serviceAccountName"] = string(pod.Spec.ServiceAccountName)
	podSpecificFieldsSet["status.phase"] = string(pod.Status.Phase)
	// TODO: add podIPs as a downward API value(s) with proper format
	podIP := ""
	if len(pod.Status.PodIPs) > 0 {
		podIP = string(pod.Status.PodIPs[0].IP)
	}
	podSpecificFieldsSet["status.podIP"] = podIP
	podSpecificFieldsSet["status.nominatedNodeName"] = string(pod.Status.NominatedNodeName)
	return generic.AddObjectMetaFieldsSet(podSpecificFieldsSet, &pod.ObjectMeta, true)
}

// 再添加上name和namespace
func AddObjectMetaFieldsSet(source fields.Set, objectMeta *metav1.ObjectMeta, hasNamespaceField bool) fields.Set {
	source["metadata.name"] = objectMeta.Name
	if hasNamespaceField {
		source["metadata.namespace"] = objectMeta.Namespace
	}
	return source
}
```
