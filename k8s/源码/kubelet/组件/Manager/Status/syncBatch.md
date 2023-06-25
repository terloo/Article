# syncBatch
syncBatch为StatusManager定时运行的同步方法，检查缓存是否合法，Pod是否还存在等

## syncBatch
```go
// pkg/kubelet/status/status_manager.go
func (m *manager) syncBatch() {
	var updatedStatuses []podStatusSyncRequest
	podToMirror, mirrorToPod := m.podManager.GetUIDTranslations()
	func() { // Critical section
		m.podStatusesLock.RLock()
		defer m.podStatusesLock.RUnlock()

		// Clean up orphaned versions.
		// 清理apiStatusVersions中有，而podStatuses没有的versionedPodStatus
		for uid := range m.apiStatusVersions {
			_, hasPod := m.podStatuses[types.UID(uid)]
			_, hasMirror := mirrorToPod[uid]
			if !hasPod && !hasMirror {
				delete(m.apiStatusVersions, uid)
			}
		}

		// 清理podStatuses中有
		for uid, status := range m.podStatuses {
			syncedUID := kubetypes.MirrorPodUID(uid)
			// 获取对应的mirror pod如果是static pod
			if mirrorUID, ok := podToMirror[kubetypes.ResolvedPodUID(uid)]; ok {
				if mirrorUID == "" {
					klog.V(5).InfoS("Static pod does not have a corresponding mirror pod; skipping",
						"podUID", uid,
						"pod", klog.KRef(status.podNamespace, status.podName))
					continue
				}
				syncedUID = mirrorUID
			}
			if m.needsUpdate(types.UID(syncedUID), status) {
				// 主要是判断Pod是否资源回收完毕
				updatedStatuses = append(updatedStatuses, podStatusSyncRequest{uid, status})
			} else if m.needsReconcile(uid, status.status) {
				// Delete the apiStatusVersions here to force an update on the pod status
				// In most cases the deleted apiStatusVersions here should be filled
				// soon after the following syncPod() [If the syncPod() sync an update
				// successfully].
				delete(m.apiStatusVersions, syncedUID)
				updatedStatuses = append(updatedStatuses, podStatusSyncRequest{uid, status})
			}
		}
	}()

	for _, update := range updatedStatuses {
		klog.V(5).InfoS("Status Manager: syncPod in syncbatch", "podUID", update.podUID)
		m.syncPod(update.podUID, update.status)
	}
}
```

## needsReconcile
对比当前状态与PodManager中状态是否一致
```go
// pkg/kubelet/stauts/status_manager.go
func (m *manager) needsReconcile(uid types.UID, status v1.PodStatus) bool {
	// The pod could be a static pod, so we should translate first.
	pod, ok := m.podManager.GetPodByUID(uid)
	if !ok {
		klog.V(4).InfoS("Pod has been deleted, no need to reconcile", "podUID", string(uid))
		return false
	}
	// If the pod is a static pod, we should check its mirror pod, because only status in mirror pod is meaningful to us.
	if kubetypes.IsStaticPod(pod) {
		mirrorPod, ok := m.podManager.GetMirrorPodByPod(pod)
		if !ok {
			klog.V(4).InfoS("Static pod has no corresponding mirror pod, no need to reconcile", "pod", klog.KObj(pod))
			return false
		}
		pod = mirrorPod
	}

	podStatus := pod.Status.DeepCopy()
	normalizeStatus(pod, podStatus)

	if isPodStatusByKubeletEqual(podStatus, &status) {
		// If the status from the source is the same with the cached status,
		// reconcile is not needed. Just return.
		return false
	}
	klog.V(3).InfoS("Pod status is inconsistent with cached status for pod, a reconciliation should be triggered",
		"pod", klog.KObj(pod),
		"statusDiff", diff.ObjectDiff(podStatus, &status))

	return true
}
```