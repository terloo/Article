# PLEG
即`Pod Lifecycle Event Generator`，用于处理Pod资源的被动性改动  
PLEG会定期运行relist方法以检查节点上Pod的运行情况，并将pod的变化包装成事件，传递给syncLoop进行处理

## 间隔
1. 检查的执行间隔：每一次检查过程后，PLEG会等待1s，再次进行检查
2. 检查的超时时间：如果一次检查不能在3min内完成，NodeStatus会认为此节点Not Ready


## 结构体
```go
// pkg/kubelet/pleg/generic.go
type GenericPLEG struct {
	// The period for relisting.
	relistPeriod time.Duration
	// The container runtime.
	runtime kubecontainer.Runtime
	// The channel from which the subscriber listens events.
	eventChannel chan *PodLifecycleEvent
	// The internal cache for pod/container information.
	podRecords podRecords
	// Time of the last relisting.
	relistTime atomic.Value
	// Cache for storing the runtime states required for syncing pods.
	cache kubecontainer.Cache
	// For testability.
	clock clock.Clock
	// Pods that failed to have their status retrieved during a relist. These pods will be
	// retried during the next relisting.
	podsToReinspect map[types.UID]*kubecontainer.Pod
}
```
1. relistPeriod：relist方法执行的时间间隔
2. eventChannel：PodLifecycleEvent的事件推送channel
3. podRecords：记录pod当前状态和上一次的状态
4. relistTime：最近一次relist的事件
5. cache：缓存Pod的详细状态
6. podsToReinspect：记录获取详细状态失败的Pod

## 构建函数
```go
// pkg/kubelet/pleg/generic.go
func NewGenericPLEG(runtime kubecontainer.Runtime, channelCapacity int,
	relistPeriod time.Duration, cache kubecontainer.Cache, clock clock.Clock) PodLifecycleEventGenerator {
	return &GenericPLEG{
		relistPeriod: relistPeriod,
		runtime:      runtime,
		eventChannel: make(chan *PodLifecycleEvent, channelCapacity),
		podRecords:   make(podRecords),
		cache:        cache,
		clock:        clock,
	}
}
```

## Watch
返回PLEG的PodLifecycleEvent事件channel
```go
// pkg/kubelet/pleg/generic.go
func (g *GenericPLEG) Watch() chan *PodLifecycleEvent {
	return g.eventChannel
}
```

## Start
启动PLEG，以relistPeriod的间隔开始relist循环
```go
// pkg/kubelet/pleg/generic.go
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}
```

## computeEvents
获取一个container在新旧两种Pod状态中的差别，并根据此生成事件
```go
// pkg/kubelet/pleg/generic.go
func computeEvents(oldPod, newPod *kubecontainer.Pod, cid *kubecontainer.ContainerID) []*PodLifecycleEvent {
	var pid types.UID
	// 如果有旧状态，则使用旧Pod的id进行通知，否则使用新状态的Pod的id进行通知
	// 实际上这两个id一般是同一个
	if oldPod != nil {
		pid = oldPod.ID
	} else if newPod != nil {
		pid = newPod.ID
	}
	oldState := getContainerState(oldPod, cid)
	newState := getContainerState(newPod, cid)
	// 根据新老状态生成PodLifycycleEvent
	return generateEvents(pid, cid.ID, oldState, newState)
}

// 遍历pod的container和sandbox，获取该containerId在pod中的状态
func getContainerState(pod *kubecontainer.Pod, cid *kubecontainer.ContainerID) plegContainerState {
	// Default to the non-existent state.
	// 默认状态是non-existent(不存在)
	state := plegContainerNonExistent
	if pod == nil {
		return state
	}
	c := pod.FindContainerByID(*cid)
	if c != nil {
		// 将原生的container状态转化为PLEG状态
		return convertState(c.State)
	}
	// Search through sandboxes too.
	c = pod.FindSandboxByID(*cid)
	if c != nil {
		return convertState(c.State)
	}

	return state
}
```

## updateCache
检查并更新pod的详细状态，如果检查状态失败，将err保存到缓存中并返回
```go
// pkg/kubelet/pleg/generic.go
func (g *GenericPLEG) updateCache(pod *kubecontainer.Pod, pid types.UID) error {
	if pod == nil {
		// The pod is missing in the current relist. This means that
		// the pod has no visible (active or inactive) containers.
		klog.V(4).InfoS("PLEG: Delete status for pod", "podUID", string(pid))
		g.cache.Delete(pid)
		return nil
	}
	timestamp := g.clock.Now()
	// TODO: Consider adding a new runtime method
	// GetPodStatus(pod *kubecontainer.Pod) so that Docker can avoid listing
	// all containers again.
	status, err := g.runtime.GetPodStatus(pod.ID, pod.Name, pod.Namespace)
	if klog.V(6).Enabled() {
		klog.V(6).ErrorS(err, "PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name), "podStatus", status)
	} else {
		klog.V(4).ErrorS(err, "PLEG: Write status", "pod", klog.KRef(pod.Namespace, pod.Name))
	}
	if err == nil {
		// Preserve the pod IP across cache updates if the new IP is empty.
		// When a pod is torn down, kubelet may race with PLEG and retrieve
		// a pod status after network teardown, but the kubernetes API expects
		// the completed pod's IP to be available after the pod is dead.
		// 获取IP，如果pod以退出，则以缓存中的Pod详细状态为准
		status.IPs = g.getPodIPs(pid, status)
	}

	g.cache.Set(pod.ID, status, err, timestamp)
	return err
}
```

## relist
PLEG的核心方法，从container runtime中查询pod/containers，与内部缓存相比较，生成相应的事件
```go
// pkg/kubelet/pleg/generic.go
func (g *GenericPLEG) relist() {
	klog.V(5).InfoS("GenericPLEG: Relisting")

	// 将上一次relist的触发时间应用到metrics中
	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInSeconds(lastRelistTime))
	}

	// defer 这一次relist的时间应用到metric中
	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistDuration.Observe(metrics.SinceInSeconds(timestamp))
	}()

	// Get all the pods.
	// 从CRI中获得所有Pod/containers
	podList, err := g.runtime.GetPods(true)
	if err != nil {
		klog.ErrorS(err, "GenericPLEG: Unable to retrieve pods")
		return
	}

	// 更新relist时间
	g.updateRelistTime(timestamp)

	pods := kubecontainer.Pods(podList)
	// update running pod and container count
	// 将当前pod和container的状态更新到metric
	updateRunningPodAndContainerMetrics(pods)
	// 更新podRecords的current状态
	g.podRecords.setCurrent(pods)

	// Compare the old and the current pods, and generate events.
	// 比较old和current的状态，生成PodLifecycleEvent
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		// Get all containers in the old and the new pod.
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
			// 计算Pod中所有的container的变化事件
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				// 将事件添加到eventsByPodID数据结构当中
				updateEvents(eventsByPodID, e)
			}
		}
	}

	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

	// If there are events associated with a pod, we should update the
	// podCache.
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			// 检查并更新Pod详细状态缓存，如果检查失败，加入到needsReinspection中
			// 并且继续循环，不更新podRecords也不进行事件通知
			if err := g.updateCache(pod, pid); err != nil {
				// Rely on updateCache calling GetPodStatus to log the actual error.
				klog.V(4).ErrorS(err, "PLEG: Ignoring events for pod", "pod", klog.KRef(pod.Namespace, pod.Name))

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else {
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
			}
		}
		// Update the internal storage and send out the events.
		// 将current设置到old中
		g.podRecords.update(pid)

		// Map from containerId to exit code; used as a temporary cache for lookup
		containerExitCode := make(map[string]int)

		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
				// 进行事件通知
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.Inc()
				klog.ErrorS(nil, "Event channel is full, discard this relist() cycle event")
			}
			// Log exit code of containers when they finished in a particular event
			// 记录一下已完成Lifecycle的Pod
			if events[i].Type == ContainerDied {
				// Fill up containerExitCode map for ContainerDied event when first time appeared
				if len(containerExitCode) == 0 && pod != nil && g.cache != nil {
					// Get updated podStatus
					status, err := g.cache.Get(pod.ID)
					if err == nil {
						for _, containerStatus := range status.ContainerStatuses {
							containerExitCode[containerStatus.ID.ID] = containerStatus.ExitCode
						}
					}
				}
				if containerID, ok := events[i].Data.(string); ok {
					if exitCode, ok := containerExitCode[containerID]; ok && pod != nil {
						klog.V(2).InfoS("Generic (PLEG): container finished", "podID", pod.ID, "containerID", containerID, "exitCode", exitCode)
					}
				}
			}
		}
	}

	// 重新检查上次检查状态失败的Pod
	if g.cacheEnabled() {
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			klog.V(5).InfoS("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
				if err := g.updateCache(pod, pid); err != nil {
					// Rely on updateCache calling GetPodStatus to log the actual error.
					klog.V(5).ErrorS(err, "PLEG: pod failed reinspection", "pod", klog.KRef(pod.Namespace, pod.Name))
					needsReinspection[pid] = pod
				}
			}
		}

		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		g.cache.UpdateTime(timestamp)
	}

	// make sure we retain the list of pods that need reinspecting the next time relist is called
	// 将本次检查失败的Pod暂存到podsToReinspect
	g.podsToReinspect = needsReinspection
}
```

## Healthy
判断
```go
// pkg/kubelet/pleg/generic.go

const relistThreshold = 3 * time.Minute

func (g *GenericPLEG) Healthy() (bool, error) {
	// 获取relist的执行时间
	relistTime := g.getRelistTime()
	if relistTime.IsZero() {
		return false, fmt.Errorf("pleg has yet to be successful")
	}
	// Expose as metric so you can alert on `time()-pleg_last_seen_seconds > nn`
	metrics.PLEGLastSeen.Set(float64(relistTime.Unix()))
	// 计算relist执行时间与当前时间的差距，判断差距是否大于relistThreshold
	// 如果大于的话，说明本次relist超时了
	elapsed := g.clock.Since(relistTime)
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
	return true, nil
}
```