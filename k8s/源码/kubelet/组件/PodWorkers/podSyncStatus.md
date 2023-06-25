# podSyncStatus
podSyncStatus是podWorkers中需要通知给worker goroutine的信息集合

## 结构体
```go
type podSyncStatus struct {
	// ctx is the context that is associated with the current pod sync.
	ctx context.Context
	// cancelFn if set is expected to cancel the current sync*Pod operation.
	cancelFn context.CancelFunc
	// working is true if a pod worker is currently in a sync method.
	working bool
	// fullname of the pod
	fullname string

	// syncedAt is the time at which the pod worker first observed this pod.
	syncedAt time.Time
	// terminatingAt is set once the pod is requested to be killed - note that
	// this can be set before the pod worker starts terminating the pod, see
	// terminating.
	terminatingAt time.Time
	// startedTerminating is true once the pod worker has observed the request to
	// stop a pod (exited syncPod and observed a podWork with WorkType
	// TerminatingPodWork). Once this is set, it is safe for other components
	// of the kubelet to assume that no other containers may be started.
	startedTerminating bool
	// deleted is true if the pod has been marked for deletion on the apiserver
	// or has no configuration represented (was deleted before).
	deleted bool
	// gracePeriod is the requested gracePeriod once terminatingAt is nonzero.
	gracePeriod int64
	// evicted is true if the kill indicated this was an eviction (an evicted
	// pod can be more aggressively cleaned up).
	evicted bool
	// terminatedAt is set once the pod worker has completed a successful
	// syncTerminatingPod call and means all running containers are stopped.
	terminatedAt time.Time
	// finished is true once the pod worker completes for a pod
	// (syncTerminatedPod exited with no errors) until SyncKnownPods is invoked
	// to remove the pod. A terminal pod (Succeeded/Failed) will have
	// termination status until the pod is deleted.
	finished bool
	// restartRequested is true if the pod worker was informed the pod is
	// expected to exist (update type of create, update, or sync) after
	// it has been killed. When known pods are synced, any pod that is
	// terminated and has restartRequested will have its history cleared.
	restartRequested bool
	// notifyPostTerminating will be closed once the pod transitions to
	// terminated. After the pod is in terminated state, nothing should be
	// added to this list.
	notifyPostTerminating []chan<- struct{}
	// statusPostTerminating is a list of the status changes associated
	// with kill pod requests. After the pod is in terminated state, nothing
	// should be added to this list. The worker will execute the last function
	// in this list on each termination attempt.
	statusPostTerminating []PodStatusFunc
}
```
1. working：该pod对应的work goroutine是否还在处理一个work
2. syncedAt：PodWorkers第一次收到该Pod的Work，生成该缓存的时间
3. terminatingAt：第一次收到或判断出Pod应该开始终止操作的时间，注意此时Woker还未正式开始处理本次work
4. startedTerminating：Woker正式开始处理该Pod的终止请求
5. deleted：当Pod是被其来源删除时(不存在于来源config中)，字段被设置为true
6. gracePeriod：优雅删除的间隔时间，terminationAt不为0时生效
7. evicted：表示是否是驱逐操作
8. terminatedAt：syncTerminatingPod方法调用结束时时间，意味着pod的所有容器都被停止
9. finished：syncTerminatedPod方法无错退出后被设置为true

## 方法
```go
// pkg/kubelet/pod_workers.go
func (s *podSyncStatus) IsWorking() bool              { return s.working }
func (s *podSyncStatus) IsTerminationRequested() bool { return !s.terminatingAt.IsZero() }
func (s *podSyncStatus) IsTerminationStarted() bool   { return s.startedTerminating }
func (s *podSyncStatus) IsTerminated() bool           { return !s.terminatedAt.IsZero() }
func (s *podSyncStatus) IsFinished() bool             { return s.finished }
func (s *podSyncStatus) IsEvicted() bool              { return s.evicted }
func (s *podSyncStatus) IsDeleted() bool              { return s.deleted }
```