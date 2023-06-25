# podRecords
podRecord是PLEG中用于保存pod/containers信息的缓存数据结构

## 结构体
```go
// pkg/kubelet/pleg/generic.go
type podRecords map[types.UID]*podRecord

type podRecord struct {
	old     *kubecontainer.Pod
	current *kubecontainer.Pod
}
```

## 方法
```go
// pkg/kubelet/pleg/generic.go
// 返回某UID的Pod的旧状态
func (pr podRecords) getOld(id types.UID) *kubecontainer.Pod {
	r, ok := pr[id]
	if !ok {
		return nil
	}
	return r.old
}

// 返回某UID的Pod的当前状态
func (pr podRecords) getCurrent(id types.UID) *kubecontainer.Pod {
	r, ok := pr[id]
	if !ok {
		return nil
	}
	return r.current
}

// 设置所有pod的当前状态
func (pr podRecords) setCurrent(pods []*kubecontainer.Pod) {
    // 先将当前所有缓存的当前状态清空
	for i := range pr {
		pr[i].current = nil
	}
    // 再进行设置
	for _, pod := range pods {
		if r, ok := pr[pod.ID]; ok {
			r.current = pod
		} else {
			pr[pod.ID] = &podRecord{current: pod}
		}
	}
}

// 将某个UID的pod的当前状态刷新为旧状态
func (pr podRecords) update(id types.UID) {
	r, ok := pr[id]
	if !ok {
		return
	}
	pr.updateInternal(id, r)
}

func (pr podRecords) updateInternal(id types.UID, r *podRecord) {
    // 如果没有当前状态，则表明pod已被删除
	if r.current == nil {
		// Pod no longer exists; delete the entry.
		delete(pr, id)
		return
	}
	r.old = r.current
	r.current = nil
}
```