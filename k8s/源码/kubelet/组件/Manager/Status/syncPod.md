# syncPod
将Pod状态同步到api-server，在更新时，还会判断pod能否移除，如果能移除，则从ETCD中移除Pod(不包括mirror pod)

## syncPod
将某个Pod的状态同步到api-server
```go
// kubelet/status/status_manager.go
func (m *manager) syncPod(uid types.UID, status versionedPodStatus) {
	if !m.needsUpdate(uid, status) {
		klog.V(1).InfoS("Status for pod is up-to-date; skipping", "podUID", uid)
		return
	}

	// TODO: make me easier to express from client code
	pod, err := m.kubeClient.CoreV1().Pods(status.podNamespace).Get(context.TODO(), status.podName, metav1.GetOptions{})
	// 如果pod已经被移除，则忽略该次改动
	if errors.IsNotFound(err) {
		klog.V(3).InfoS("Pod does not exist on the server",
			"podUID", uid,
			"pod", klog.KRef(status.podNamespace, status.podName))
		// If the Pod is deleted the status will be cleared in
		// RemoveOrphanedStatuses, so we just ignore the update here.
		return
	}
	if err != nil {
		klog.InfoS("Failed to get status for pod",
			"podUID", uid,
			"pod", klog.KRef(status.podNamespace, status.podName),
			"err", err)
		return
	}

	// 如果pod是mirror pod，获取其static pod的uid
	translatedUID := m.podManager.TranslatePodUID(pod.UID)
	// Type convert original uid just for the purpose of comparison.
	if len(translatedUID) > 0 && translatedUID != kubetypes.ResolvedPodUID(uid) {
		// uid不同说明pod被删除并重建了，直接清理掉源uid的pod状态即可
		klog.V(2).InfoS("Pod was deleted and then recreated, skipping status update",
			"pod", klog.KObj(pod),
			"oldPodUID", uid,
			"podUID", translatedUID)
		m.deletePodStatus(uid)
		return
	}

	// 从api-server获取到的状态是旧状态
	oldStatus := pod.Status.DeepCopy()
	// 将旧状态与新状态合并并更新
	newPod, patchBytes, unchanged, err := statusutil.PatchPodStatus(m.kubeClient, pod.Namespace, pod.Name, pod.UID, *oldStatus, mergePodStatus(*oldStatus, status.status))
	klog.V(3).InfoS("Patch status for pod", "pod", klog.KObj(pod), "patch", string(patchBytes))

	if err != nil {
		klog.InfoS("Failed to update status for pod", "pod", klog.KObj(pod), "err", err)
		return
	}
	if unchanged {
		klog.V(3).InfoS("Status for pod is up-to-date", "pod", klog.KObj(pod), "statusVersion", status.version)
	} else {
		klog.V(3).InfoS("Status for pod updated successfully", "pod", klog.KObj(pod), "statusVersion", status.version, "status", status.status)
		pod = newPod
	}

	m.apiStatusVersions[kubetypes.MirrorPodUID(pod.UID)] = status.version

	// We don't handle graceful deletion of mirror pods.
	if m.canBeDeleted(pod, status.status) {
		// 发送一个GracePeriodSeconds为0的DELETE请求，ETCD收到该请求后直接删除
		deleteOptions := metav1.DeleteOptions{
			GracePeriodSeconds: new(int64),
			// Use the pod UID as the precondition for deletion to prevent deleting a
			// newly created pod with the same name and namespace.
			Preconditions: metav1.NewUIDPreconditions(string(pod.UID)),
		}
		err = m.kubeClient.CoreV1().Pods(pod.Namespace).Delete(context.TODO(), pod.Name, deleteOptions)
		if err != nil {
			klog.InfoS("Failed to delete status for pod", "pod", klog.KObj(pod), "err", err)
			return
		}
		klog.V(3).InfoS("Pod fully terminated and removed from etcd", "pod", klog.KObj(pod))
		m.deletePodStatus(uid)
	}
}
```

## needsUpdate
判断该状态是否应该被同步
```go
// pkg/kubelet/status/status_manager.go
func (m *manager) needsUpdate(uid types.UID, status versionedPodStatus) bool {
	latest, ok := m.apiStatusVersions[kubetypes.MirrorPodUID(uid)]
	// 如果该状态的版本号大于最近一次的同步版本号，则同步
	if !ok || latest < status.version {
		return true
	}
	pod, ok := m.podManager.GetPodByUID(uid)
	if !ok {
		return false
	}
	// 如果版本较低，则判断该pod资源是否已经被回收完毕，如果回收完毕，则允许同步(移除)
	return m.canBeDeleted(pod, status.status)
}
```

## canBeDeleted
判断Pod能否被移除
```go
// pkg/kubelet/status/status_manager.go
func (m *manager) canBeDeleted(pod *v1.Pod, status v1.PodStatus) bool {
    // 不移除没有接到优雅删除请求的Pod
    // 不移除mirror pod
	if pod.DeletionTimestamp == nil || kubetypes.IsMirrorPod(pod) {
		return false
	}
    // 不移除资源未释放的Pod
	return m.podDeletionSafety.PodResourcesAreReclaimed(pod, status)
}
```