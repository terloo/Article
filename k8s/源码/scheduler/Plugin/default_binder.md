# default_binder
default_binder是k8s中默认的Bind阶段的插件，用于绑定Pod和Node

## 结构体
```go
// pkg/scheduler/framework/plugins/defaultbinder/default_binder.go
type DefaultBinder struct {
    // 实际实现类是frameworkImple
	handle framework.Handle
}
```

## Bind
```go
// pkg/scheduler/framework/plugins/defaultbinder/default_binder.go
// Bind binds pods to nodes using the k8s client.
func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p *v1.Pod, nodeName string) *framework.Status {
	klog.V(3).InfoS("Attempting to bind pod to node", "pod", klog.KObj(p), "node", nodeName)
	binding := &v1.Binding{
		ObjectMeta: metav1.ObjectMeta{Namespace: p.Namespace, Name: p.Name, UID: p.UID},
		Target:     v1.ObjectReference{Kind: "Node", Name: nodeName},
	}
    // 使用ClentSet api来绑定Pod
	err := b.handle.ClientSet().CoreV1().Pods(binding.Namespace).Bind(ctx, binding, metav1.CreateOptions{})
	if err != nil {
		return framework.AsStatus(err)
	}
	return nil
}
```