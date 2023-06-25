# UpdatePodOptions
UpdatePodOptions是传递给UpdatePod的保存Pod更新信息的结构体

## 结构体
```go
// pkg/kubelet/pod_workers.go
type UpdatePodOptions struct {
	// The type of update (create, update, sync, kill).
	UpdateType kubetypes.SyncPodType
	// StartTime is an optional timestamp for when this update was created. If set,
	// when this update is fully realized by the pod worker it will be recorded in
	// the PodWorkerDuration metric.
	StartTime time.Time
	// Pod to update. Required.
	Pod *v1.Pod
	// MirrorPod is the mirror pod if Pod is a static pod. Optional when UpdateType
	// is kill or terminated.
	MirrorPod *v1.Pod
	// RunningPod is a runtime pod that is no longer present in config. Required
	// if Pod is nil, ignored if Pod is set.
	RunningPod *kubecontainer.Pod
	// KillPodOptions is used to override the default termination behavior of the
	// pod or to update the pod status after an operation is completed. Since a
	// pod can be killed for multiple reasons, PodStatusFunc is invoked in order
	// and later kills have an opportunity to override the status (i.e. a preemption
	// may be later turned into an eviction).
	KillPodOptions *KillPodOptions
}
```
1. UpdateType：更新类型
2. StartTime：该UpdatePodOptions被创建出来时的时间戳，会被用于计算PodWorkerDuration指标
3. Pod：需要进行更新的Pod
4. MirrorPod：如果Pod是静态pod，保存pod对应的mirrorPod，UpdateType是kill或terminated时可选
5. RunningPod：是PodConfig中不存在，但实际存在的Pod。与Pod字段有且只有一个为nil
6. KillPodOptions：用于覆盖默认的Pod终止行为或者在操作完成后更新Pod状态

## SyncPodType
```go
// pkg/kubelet/types/pod_update.go
type SyncPodType int

const (
	// SyncPodSync is when the pod is synced to ensure desired state
	SyncPodSync SyncPodType = iota
	// SyncPodUpdate is when the pod is updated from source
	SyncPodUpdate
	// SyncPodCreate is when the pod is created from source
	SyncPodCreate
	// SyncPodKill is when the pod should have no running containers. A pod stopped in this way could be
	// restarted in the future due config changes.
	SyncPodKill
)
```
1. SyncPodSync：同步Pod到期望的状态
2. SyncPodUpdate：Pod被source所更新
3. SyncPodCreate：Pod被source所创建
4. SyncPodKill：Pod被删除