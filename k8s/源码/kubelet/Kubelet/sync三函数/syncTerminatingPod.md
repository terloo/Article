# syncTerminatingPod
syncTerminatingPod是用于执行Pod停止操作的同步函数

## 流程
1. 处理孤儿Pod
2. 根据containerPodStatus生成apiPodStatus
3. 更新状态到apiServer
4. 调用CRI同步接口来停止容器
5. 停止后查询状态，如果还有运行中的容器，打印日志

## syncTerminatingPod
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncTerminatingPod(ctx context.Context, pod *v1.Pod, podStatus *kubecontainer.PodStatus, runningPod *kubecontainer.Pod, gracePeriod *int64, podStatusFn func(*v1.PodStatus)) error {
	klog.V(4).InfoS("syncTerminatingPod enter", "pod", klog.KObj(pod), "podUID", pod.UID)
	defer klog.V(4).InfoS("syncTerminatingPod exit", "pod", klog.KObj(pod), "podUID", pod.UID)

	// when we receive a runtime only pod (runningPod != nil) we don't need to update the status
	// manager or refresh the status of the cache, because a successful killPod will ensure we do
	// not get invoked again
	if runningPod != nil {
		// we kill the pod with the specified grace period since this is a termination
		if gracePeriod != nil {
			klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", *gracePeriod)
		} else {
			klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", nil)
		}
		if err := kl.killPod(pod, *runningPod, gracePeriod); err != nil {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
			// there was an error killing the pod, so we return that error directly
			utilruntime.HandleError(err)
			return err
		}
		klog.V(4).InfoS("Pod termination stopped all running orphan containers", "pod", klog.KObj(pod), "podUID", pod.UID)
		return nil
	}

	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	if podStatusFn != nil {
		podStatusFn(&apiPodStatus)
	}
	kl.statusManager.SetPodStatus(pod, apiPodStatus)

	if gracePeriod != nil {
		klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", *gracePeriod)
	} else {
		klog.V(4).InfoS("Pod terminating with grace period", "pod", klog.KObj(pod), "podUID", pod.UID, "gracePeriod", nil)
	}
	p := kubecontainer.ConvertPodStatusToRunningPod(kl.getRuntime().Type(), podStatus)
	if err := kl.killPod(pod, p, gracePeriod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToKillPod, "error killing pod: %v", err)
		// there was an error killing the pod, so we return that error directly
		utilruntime.HandleError(err)
		return err
	}

	// Guard against consistency issues in KillPod implementations by checking that there are no
	// running containers. This method is invoked infrequently so this is effectively free and can
	// catch race conditions introduced by callers updating pod status out of order.
	// TODO: have KillPod return the terminal status of stopped containers and write that into the
	//  cache immediately
	podStatus, err := kl.containerRuntime.GetPodStatus(pod.UID, pod.Name, pod.Namespace)
	if err != nil {
		klog.ErrorS(err, "Unable to read pod status prior to final pod termination", "pod", klog.KObj(pod), "podUID", pod.UID)
		return err
	}
	var runningContainers []string
	var containers []string
	for _, s := range podStatus.ContainerStatuses {
		if s.State == kubecontainer.ContainerStateRunning {
			runningContainers = append(runningContainers, s.ID.String())
		}
		containers = append(containers, fmt.Sprintf("(%s state=%s exitCode=%d finishedAt=%s)", s.Name, s.State, s.ExitCode, s.FinishedAt.UTC().Format(time.RFC3339Nano)))
	}
	if klog.V(4).Enabled() {
		sort.Strings(containers)
		klog.InfoS("Post-termination container state", "pod", klog.KObj(pod), "podUID", pod.UID, "containers", strings.Join(containers, " "))
	}
	if len(runningContainers) > 0 {
		return fmt.Errorf("detected running containers after a successful KillPod, CRI violation: %v", runningContainers)
	}

	// we have successfully stopped all containers, the pod is terminating, our status is "done"
	klog.V(4).InfoS("Pod termination stopped all running containers", "pod", klog.KObj(pod), "podUID", pod.UID)

	return nil
}
```