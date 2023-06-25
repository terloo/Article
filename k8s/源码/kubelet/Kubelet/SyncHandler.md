# SyncHandler

## 接口
```go
// pkg/kubelet/kubelet.go
type SyncHandler interface {
	HandlePodAdditions(pods []*v1.Pod)
	HandlePodUpdates(pods []*v1.Pod)
	HandlePodRemoves(pods []*v1.Pod)
	HandlePodReconcile(pods []*v1.Pod)
	HandlePodSyncs(pods []*v1.Pod)
	HandlePodCleanups() error
}
```

## HandlePodAdditions
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
        // 添加到podManager缓存
		kl.podManager.AddPod(pod)

		if kubetypes.IsMirrorPod(pod) {
			// 如果是mirrorPod，处理mirroPod
			kl.handleMirrorPod(pod, start)
			continue
		}

		// Only go through the admission process if the pod is not requested
		// for termination by another part of the kubelet. If the pod is already
		// using resources (previously admitted), the pod worker is going to be
		// shutting it down. If the pod hasn't started yet, we know that when
		// the pod worker is invoked it will also avoid setting up the pod, so
		// we simply avoid doing any work.
		if !kl.podWorkers.IsPodTerminationRequested(pod.UID) {
			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			// 去除已经终止的Pod
			activePods := kl.filterOutInactivePods(existingPods)

			// Check if we can admit the pod; if not, reject it.
			// 如果该节点不能承载该Pod，则拒绝
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		// 将任务分发给
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
		// TODO: move inside syncPod and make reentrant
		// https://github.com/kubernetes/kubernetes/issues/105014
		kl.probeManager.AddPod(pod)
	}
}
```