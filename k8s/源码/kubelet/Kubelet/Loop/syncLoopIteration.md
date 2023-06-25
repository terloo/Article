# syncLoopIteration
随机从各种渠道中的一个接收事件，将其分发到handler参数中进行处理，并更新monitor时间戳
| 渠道           | 事件                                | 处理方法                                        |
| -------------- | ----------------------------------- | ----------------------------------------------- |
| configCh       | Pod配置事件(HTTP/静态Pod/apiServer) | 分发到适当的handler回调中                       |
| plegCh         | PLEG事件                            | 更新runtime缓存，同步Pod                        |
| syncCh         | 周期性同步Ticker                    | 同步等待同步的Pod                               |
| housekeepingCh | 周期性强制循环Ticker                | 触发Pod的cleanup                                |
| health manager | 健康检查事件                        | 同步健康检查失败或其中一个容器健康检查失败的Pod |
> health manager包括了`livenessManager`、`readinessManager`、`startupManager`

## configCh循环
直接调用handler对应的方法进行处理，不处理SET。
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			klog.ErrorS(nil, "Update channel is closed, exiting the sync loop")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			klog.V(2).InfoS("SyncLoop ADD", "source", u.Source, "pods", klog.KObjs(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			klog.V(2).InfoS("SyncLoop UPDATE", "source", u.Source, "pods", klog.KObjs(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			klog.V(2).InfoS("SyncLoop REMOVE", "source", u.Source, "pods", klog.KObjs(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			klog.V(4).InfoS("SyncLoop RECONCILE", "source", u.Source, "pods", klog.KObjs(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			klog.V(2).InfoS("SyncLoop DELETE", "source", u.Source, "pods", klog.KObjs(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			// 由于需要优雅删除，所以DELETE操作被视为更新
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			klog.ErrorS(nil, "Kubelet does not support snapshot update")
		default:
			klog.ErrorS(nil, "Invalid operation type received", "operation", u.Op)
		}

		// 向sourcesReady中添加该来源，这样sourcesReady即可进行健康检查
		kl.sourcesReady.AddSource(u.Source)
    }

    // ... case其他channel
}
```

## plegCh
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {

    // ... case其他channel

	case e := <-plegCh:
	if e.Type == pleg.ContainerStarted {
		// record the most recent time we observed a container start for this pod.
		// this lets us selectively invalidate the runtimeCache when processing a delete for this pod
		// to make sure we don't miss handling graceful termination for containers we reported as having started.
		// 如果是ContainerStarted事件，需要额外记录一下该Pod最近一次容器启动时间
		kl.lastContainerStartedTime.Add(e.ID, time.Now())
	}
	// Removed事件发生时，不进行Pod同步
	if isSyncPodWorthy(e) {
		// PLEG event for a pod; sync it.
		// 如果podManager能找到该Pod，则进行Pod同步
		if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
			klog.V(2).InfoS("SyncLoop (PLEG): event for pod", "pod", klog.KObj(pod), "event", e)
			// 进行Pod同步
			handler.HandlePodSyncs([]*v1.Pod{pod})
		} else {
			// If the pod no longer exists, ignore the event.
			klog.V(4).InfoS("SyncLoop (PLEG): pod does not exist, ignore irrelevant event", "event", e)
		}
	}

	// 如果是ContainerDied事件，还需要清理Pod
	if e.Type == pleg.ContainerDied {
		if containerID, ok := e.Data.(string); ok {
			kl.cleanUpContainersInPod(e.ID, containerID)
		}
	}

	// ... case其他channel
}

func isSyncPodWorthy(event *pleg.PodLifecycleEvent) bool {
	// ContainerRemoved doesn't affect pod state
	return event.Type != pleg.ContainerRemoved
}

func (kl *Kubelet) cleanUpContainersInPod(podID types.UID, exitedContainerID string) {
	if podStatus, err := kl.podCache.Get(podID); err == nil {
		// When an evicted or deleted pod has already synced, all containers can be removed.
		// Pod的容器退出，如果这个Pod已经被Kubelet同步，则需要删掉该Pod
		removeAll := kl.podWorkers.ShouldPodContentBeRemoved(podID)
		kl.containerDeletor.deleteContainersInPod(exitedContainerID, podStatus, removeAll)
	}
}
```

## syncCh
周期性的同步Ticker
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {

    // ... case其他channel

	case <-syncCh:
		// Sync pods waiting for sync
		// 获取需要同步的Pods
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		klog.V(4).InfoS("SyncLoop (SYNC) pods", "total", len(podsToSync), "pods", klog.KObjs(podsToSync))
		// 直接进行同步
		handler.HandlePodSyncs(podsToSync)

	// ... case其他channel
}
```

## health manager
探针事件
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {

    // ... case其他channel

	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			handleProbeSync(kl, update, handler, "liveness", "unhealthy")
		}
	case update := <-kl.readinessManager.Updates():
		ready := update.Result == proberesults.Success
		kl.statusManager.SetContainerReadiness(update.PodUID, update.ContainerID, ready)

		status := ""
		if ready {
			status = "ready"
		}
		handleProbeSync(kl, update, handler, "readiness", status)
	case update := <-kl.startupManager.Updates():
		started := update.Result == proberesults.Success
		kl.statusManager.SetContainerStartup(update.PodUID, update.ContainerID, started)

		status := "unhealthy"
		if started {
			status = "started"
		}
		handleProbeSync(kl, update, handler, "startup", status)

	// ... case其他channel
}

func handleProbeSync(kl *Kubelet, update proberesults.Update, handler SyncHandler, probe, status string) {
	// We should not use the pod from manager, because it is never updated after initialization.
	pod, ok := kl.podManager.GetPodByUID(update.PodUID)
	if !ok {
		// If the pod no longer exists, ignore the update.
		klog.V(4).InfoS("SyncLoop (probe): ignore irrelevant update", "probe", probe, "status", status, "update", update)
		return
	}
	klog.V(1).InfoS("SyncLoop (probe)", "probe", probe, "status", status, "pod", klog.KObj(pod))
	// 仍然是进行Syncs操作
	handler.HandlePodSyncs([]*v1.Pod{pod})
}
```

## housekeepingCh
强制性防阻塞事件，同时进行Pod清理操作
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {

    // ... case其他channel

	case <-housekeepingCh:
		// 先判断所有来源是否准备就绪，防止从未准备好的source中删除Pod
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			klog.V(4).InfoS("SyncLoop (housekeeping, skipped): sources aren't ready yet")
		} else {
			start := time.Now()
			klog.V(4).InfoS("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				klog.ErrorS(err, "Failed cleaning pods")
			}
			duration := time.Since(start)
			if duration > housekeepingWarningDuration {
				klog.ErrorS(fmt.Errorf("housekeeping took too long"), "Housekeeping took longer than 15s", "seconds", duration.Seconds())
			}
			klog.V(4).InfoS("SyncLoop (housekeeping) end")
		}

	// ... case其他channel
}
```