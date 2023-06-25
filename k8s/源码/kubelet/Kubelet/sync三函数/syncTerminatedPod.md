# syncTerminatedPod
syncTerminatedPod负责清理一个已经停止Pod

## 流程
1. 最后一次更新Pod的状态
2. 等待取消挂载
3. 移除cgroups
4. 调用statusManager的TerminatedPod方法来删除Pod资源对象

## syncTerminatedPod
```go
func (kl *Kubelet) syncTerminatedPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus) error {
	klog.V(4).InfoS("syncTerminatedPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("syncTerminatedPod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	// generate the final status of the pod
	// TODO: should we simply fold this into TerminatePod? that would give a single pod update
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	// volumes are unmounted after the pod worker reports ShouldPodRuntimeBeRemoved (which is satisfied
	// before syncTerminatedPod is invoked)
	if err := kl.volumeManager.WaitForUnmount(pod); err != nil {
		return err
	}
	klog.V(4).InfoS("Pod termination unmounted volumes", "pod", klog.KObj(pod), "podUID", pod.UID)

	// Note: we leave pod containers to be reclaimed in the background since dockershim requires the
	// container for retrieving logs and we want to make sure logs are available until the pod is
	// physically deleted.

	// remove any cgroups in the hierarchy for pods that are no longer running.
	if kl.cgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		name, _ := pcm.GetPodContainerName(pod)
		if err := pcm.Destroy(name); err != nil {
			return err
		}
		klog.V(4).InfoS("Pod termination removed cgroups", "pod", klog.KObj(pod), "podUID", pod.UID)
	}

	// mark the final pod status
	kl.statusManager.TerminatePod(pod)
	klog.V(4).InfoS("Pod is terminated and will need no more status updates", "pod", klog.KObj(pod), "podUID", pod.UID)

	return nil
}
```