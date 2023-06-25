# podWorkers
podWorkers负责根据UpdatePod的UpdatePodOptions参数和
podWorkers会给每一个Pod创建一个goroutine以及配套的channel，goroutine在收到channel中传入的有关Pod的变化时，会执行相应的逻辑来同步Pod的变化到CRI

## 注释
podWorkers会跟踪对pod的每一次操作，保证pod的状态跟container runtime和其他子系统是协调的。还会将运行中的Pod的container启动起来，将正在终止的Pod的container关闭，将已终止的Pod的container删除。  
传入podWorkers中的pod将会是以下几种状态之一：
1. synced：期望运行的Pod
2. terminating：已有运行中的container但是无新的container需要启动
3. terminated：没有正在运行的容器，但是仍然有占用资源
4. cleaned up：没有正在占用的资源
一旦一个Pod被删除或者被驱逐，将不能再次使用相同的UID进行启动，直到
1. podWorkers完成(syncTerminatingPodFnType或syncTerminatedPodFnType)
2. SyncKnownPods方法被调用，并且pod不是已知的config的一部分
podWorkers也提供了关于pod状态和容器是否能运行的持续性信息来源

## 三个关键函数 实现者还是Kubelet
1. syncPodFnType：将Kubelet管理的Pod协调到期望的状态
2. syncTerminatingPodFnType：终止Kublet管理的一个Pod
3. syncTerminatedPodFnType：清理一个已终止的Pod

## 结构体
```go
// pkg/kubelet/pod_workers.go
type podWorkers struct {
	// Protects all per worker fields.
	podLock sync.Mutex
	// podsSynced is true once the pod worker has been synced at least once,
	// which means that all working pods have been started via UpdatePod().
	podsSynced bool
	// Tracks all running per-pod goroutines - per-pod goroutine will be
	// processing updates received through its corresponding channel.
	podUpdates map[types.UID]chan podWork
	// Tracks the last undelivered work item for this pod - a work item is
	// undelivered if it comes in while the worker is working.
	lastUndeliveredWorkUpdate map[types.UID]podWork
	// Tracks by UID the termination status of a pod - syncing, terminating,
	// terminated, and evicted.
	podSyncStatuses map[types.UID]*podSyncStatus
	// Tracks all uids for started static pods by full name
	startedStaticPodsByFullname map[string]types.UID
	// Tracks all uids for static pods that are waiting to start by full name
	waitingToStartStaticPodsByFullname map[string][]types.UID

	workQueue queue.WorkQueue

	// This function is run to sync the desired state of pod.
	// NOTE: This function has to be thread-safe - it can be called for
	// different pods at the same time.

	syncPodFn            syncPodFnType
	syncTerminatingPodFn syncTerminatingPodFnType
	syncTerminatedPodFn  syncTerminatedPodFnType

	// workerChannelFn is exposed for testing to allow unit tests to impose delays
	// in channel communication. The function is invoked once each time a new worker
	// goroutine starts.
	workerChannelFn func(uid types.UID, in chan podWork) (out <-chan podWork)

	// The EventRecorder to use
	recorder record.EventRecorder

	// backOffPeriod is the duration to back off when there is a sync error.
	backOffPeriod time.Duration

	// resyncInterval is the duration to wait until the next sync.
	resyncInterval time.Duration

	// podCache stores kubecontainer.PodStatus for all pods.
	podCache kubecontainer.Cache
}
```
1. podLock：字段锁
2. syncPodFn、syncTerminatingPodFn、syncTerminatedPodFn：三函数
3. podsSynced：PodWorkers是否已经执行过了定时同步所有Pod的操作
4. podUpdates：Pod UID到worker gorountine所使用的channel的映射
5. lastUndeliveredWorkUpdate：如果上次PodWork还未执行完毕，则将这次PodWork保存到此映射
6. podSyncStatuses：保存着Pod终止状态的映射，终止状态可能为syncing、terminating、terminated、evited等
7. startedStaticPodsByFullname：staticPod到其全名的映射，主要是为StaticPod的准入过程做了个缓存
8. workQueue：kubelet的工作队列，里面保存了所有需要同步的pod。在PodWorkers对某个Pod进行操作了之后，会将该Pod放入队列中等待kubelet重新同步


## 构造函数
需要传入三个函数
```go
// pkg/kubelet/pod_workers.go
func newPodWorkers(
	syncPodFn syncPodFnType,
	syncTerminatingPodFn syncTerminatingPodFnType,
	syncTerminatedPodFn syncTerminatedPodFnType,
	recorder record.EventRecorder,
	workQueue queue.WorkQueue,
	resyncInterval, backOffPeriod time.Duration,
	podCache kubecontainer.Cache,
) PodWorkers {
	return &podWorkers{
		podSyncStatuses:                    map[types.UID]*podSyncStatus{},
		podUpdates:                         map[types.UID]chan podWork{},
		lastUndeliveredWorkUpdate:          map[types.UID]podWork{},
		startedStaticPodsByFullname:        map[string]types.UID{},
		waitingToStartStaticPodsByFullname: map[string][]types.UID{},
		syncPodFn:                          syncPodFn,
		syncTerminatingPodFn:               syncTerminatingPodFn,
		syncTerminatedPodFn:                syncTerminatedPodFn,
		recorder:                           recorder,
		workQueue:                          workQueue,
		resyncInterval:                     resyncInterval,
		backOffPeriod:                      backOffPeriod,
		podCache:                           podCache,
	}
}
```

## UpdatePod
判断Pod的状态和options的类型来获取/计算status，通过status决定WorkType，再将WorkType传入工作channel
```go
// pkg/kubelet/pod_workers.go
func (p *podWorkers) UpdatePod(options UpdatePodOptions) {
	// handle when the pod is an orphan (no config) and we only have runtime status by running only
	// the terminating part of the lifecycle
	pod := options.Pod
	var isRuntimePod bool
	// options.RunningPod字段中保存了不应该存在但实际存在于container runtime中的Pod(孤立Pod)
	if options.RunningPod != nil {
		if options.Pod == nil {
			pod = options.RunningPod.ToAPIPod()
			// 对孤立Pod，如果不是进行删除操作，则不进行任何操作
			if options.UpdateType != kubetypes.SyncPodKill {
				klog.InfoS("Pod update is ignored, runtime pods can only be killed", "pod", klog.KObj(pod), "podUID", pod.UID)
				return
			}
			options.Pod = pod
			isRuntimePod = true
		} else {
			options.RunningPod = nil
			klog.InfoS("Pod update included RunningPod which is only valid when Pod is not specified", "pod", klog.KObj(options.Pod), "podUID", options.Pod.UID)
		}
	}
	uid := pod.UID

	p.podLock.Lock()
	defer p.podLock.Unlock()

	// decide what to do with this pod - we are either setting it up, tearing it down, or ignoring it
	now := time.Now()
	status, ok := p.podSyncStatuses[uid]
	if !ok {
		// podSyncStatuses中没有保存该Pod，说明Pod是第一次被同步
		klog.V(4).InfoS("Pod is being synced for the first time", "pod", klog.KObj(pod), "podUID", pod.UID)
		status = &podSyncStatus{
			syncedAt: now,
			fullname: kubecontainer.GetPodFullName(pod),
		}
		// if this pod is being synced for the first time, we need to make sure it is an active pod
		if !isRuntimePod && (pod.Status.Phase == v1.PodFailed || pod.Status.Phase == v1.PodSucceeded) {
			// check to see if the pod is not running and the pod is terminal.
			// If this succeeds then record in the podWorker that it is terminated.
			if statusCache, err := p.podCache.Get(pod.UID); err == nil {
				if isPodStatusCacheTerminal(statusCache) {
					// 如果pod 1.资源对象还在活动状态 2.kubelet的podCache中保存的Pod状态是终止状态
					// 则将其同步状态标记为已终止
					status = &podSyncStatus{
						terminatedAt:       now,
						terminatingAt:      now,
						syncedAt:           now,
						startedTerminating: true,
						finished:           true,
						fullname:           kubecontainer.GetPodFullName(pod),
					}
				}
			}
		}
		p.podSyncStatuses[uid] = status
	}

	// if an update is received that implies the pod should be running, but we are already terminating a pod by
	// that UID, assume that two pods with the same UID were created in close temporal proximity (usually static
	// pod but it's possible for an apiserver to extremely rarely do something similar) - flag the sync status
	// to indicate that after the pod terminates it should be reset to "not running" to allow a subsequent add/update
	// to start the pod worker again
	// pod已结束，但收到了一个创建类型的同步命令
	if status.IsTerminationRequested() {
		if options.UpdateType == kubetypes.SyncPodCreate {
			status.restartRequested = true
			klog.V(4).InfoS("Pod is terminating but has been requested to restart with same UID, will be reconciled later", "pod", klog.KObj(pod), "podUID", pod.UID)
			return
		}
	}

	// once a pod is terminated by UID, it cannot reenter the pod worker (until the UID is purged by housekeeping)
	if status.IsFinished() {
		klog.V(4).InfoS("Pod is finished processing, no further updates", "pod", klog.KObj(pod), "podUID", pod.UID)
		return
	}

	// check for a transition to terminating
	// 判断pod是否应该被删除
	var becameTerminating bool
	if !status.IsTerminationRequested() {
		switch {
		case isRuntimePod:
			klog.V(4).InfoS("Pod is orphaned and must be torn down", "pod", klog.KObj(pod), "podUID", pod.UID)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.DeletionTimestamp != nil:
			klog.V(4).InfoS("Pod is marked for graceful deletion, begin teardown", "pod", klog.KObj(pod), "podUID", pod.UID)
			status.deleted = true
			status.terminatingAt = now
			becameTerminating = true
		case pod.Status.Phase == v1.PodFailed, pod.Status.Phase == v1.PodSucceeded:
			klog.V(4).InfoS("Pod is in a terminal phase (success/failed), begin teardown", "pod", klog.KObj(pod), "podUID", pod.UID)
			status.terminatingAt = now
			becameTerminating = true
		case options.UpdateType == kubetypes.SyncPodKill:
			if options.KillPodOptions != nil && options.KillPodOptions.Evict {
				klog.V(4).InfoS("Pod is being evicted by the kubelet, begin teardown", "pod", klog.KObj(pod), "podUID", pod.UID)
				status.evicted = true
			} else {
				klog.V(4).InfoS("Pod is being removed by the kubelet, begin teardown", "pod", klog.KObj(pod), "podUID", pod.UID)
			}
			status.terminatingAt = now
			becameTerminating = true
		}
	}

	// once a pod is terminating, all updates are kills and the grace period can only decrease
	var workType PodWorkType
	var wasGracePeriodShortened bool
	switch {
	case status.IsTerminated():
		// 如果Pod状态已经终止但是还在Kubelet节点上运行，说明Pod可能正在进行清理操作，忽略
		// A terminated pod may still be waiting for cleanup - if we receive a runtime pod kill request
		// due to housekeeping seeing an older cached version of the runtime pod simply ignore it until
		// after the pod worker completes.
		if isRuntimePod {
			klog.V(3).InfoS("Pod is waiting for termination, ignoring runtime-only kill until after pod worker is fully terminated", "pod", klog.KObj(pod), "podUID", pod.UID)
			return
		}

		// 工作类型为终止PodWork
		workType = TerminatedPodWork

		// 关闭CompletedCh
		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
		}
		options.KillPodOptions = nil

	case status.IsTerminationRequested():
		workType = TerminatingPodWork
		if options.KillPodOptions == nil {
			options.KillPodOptions = &KillPodOptions{}
		}

		if ch := options.KillPodOptions.CompletedCh; ch != nil {
			status.notifyPostTerminating = append(status.notifyPostTerminating, ch)
		}
		if fn := options.KillPodOptions.PodStatusFunc; fn != nil {
			status.statusPostTerminating = append(status.statusPostTerminating, fn)
		}

		// 计算优雅退出的时间间隔
		gracePeriod, gracePeriodShortened := calculateEffectiveGracePeriod(status, pod, options.KillPodOptions)

		wasGracePeriodShortened = gracePeriodShortened
		status.gracePeriod = gracePeriod
		// always set the grace period for syncTerminatingPod so we don't have to recalculate,
		// will never be zero.
		options.KillPodOptions.PodTerminationGracePeriodSecondsOverride = &gracePeriod

	default:
		// 普通的同步Pod的PodWork
		workType = SyncPodWork

		// KillPodOptions is not valid for sync actions outside of the terminating phase
		if options.KillPodOptions != nil {
			if ch := options.KillPodOptions.CompletedCh; ch != nil {
				close(ch)
			}
			options.KillPodOptions = nil
		}
	}

	// the desired work we want to be performing
	work := podWork{
		WorkType: workType,
		Options:  options,
	}

	// start the pod worker goroutine if it doesn't exist
	// 如果该pod还没有对应的PodWork goroutine，则启动一个
	podUpdates, exists := p.podUpdates[uid]
	if !exists {
		// We need to have a buffer here, because checkForUpdates() method that
		// puts an update into channel is called from the same goroutine where
		// the channel is consumed. However, it is guaranteed that in such case
		// the channel is empty, so buffer of size 1 is enough.
		podUpdates = make(chan podWork, 1)
		p.podUpdates[uid] = podUpdates

		// ensure that static pods start in the order they are received by UpdatePod
		if kubetypes.IsStaticPod(pod) {
			p.waitingToStartStaticPodsByFullname[status.fullname] =
				append(p.waitingToStartStaticPodsByFullname[status.fullname], uid)
		}

		// allow testing of delays in the pod update channel
		var outCh <-chan podWork
		if p.workerChannelFn != nil {
			outCh = p.workerChannelFn(uid, podUpdates)
		} else {
			outCh = podUpdates
		}

		// Creating a new pod worker either means this is a new pod, or that the
		// kubelet just restarted. In either case the kubelet is willing to believe
		// the status of the pod for the first pod worker sync. See corresponding
		// comment in syncPod.
		go func() {
			defer runtime.HandleCrash()
			// 启动该channel对应的消费者
			p.managePodLoop(outCh)
		}()
	}

	// dispatch a request to the pod worker if none are running
	// 
	if !status.IsWorking() {
		status.working = true
		podUpdates <- work
		return
	}

	// capture the maximum latency between a requested update and when the pod
	// worker observes it
	if undelivered, ok := p.lastUndeliveredWorkUpdate[pod.UID]; ok {
		// track the max latency between when a config change is requested and when it is realized
		// NOTE: this undercounts the latency when multiple requests are queued, but captures max latency
		if !undelivered.Options.StartTime.IsZero() && undelivered.Options.StartTime.Before(work.Options.StartTime) {
			work.Options.StartTime = undelivered.Options.StartTime
		}
	}

	// always sync the most recent data
	p.lastUndeliveredWorkUpdate[pod.UID] = work

	if (becameTerminating || wasGracePeriodShortened) && status.cancelFn != nil {
		klog.V(3).InfoS("Cancelling current pod sync", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", work.WorkType)
		status.cancelFn()
		return
	}
}
```

## managePodLoop
针对工作channel，开始进行消费，每个Pod都有一个goroutine负责消费  
根据收到WorkType决定调用sync三函数中的哪一个，根据sync三函数是否报错和WorkType进行后处理(关闭请求处理，关闭后处理，正常入队进行重同步)
```go
// pkg/kubelet/pod_workers.go
func (p *podWorkers) managePodLoop(podUpdates <-chan podWork) {
	var lastSyncTime time.Time
	var podStarted bool
	// 循环消费，channel被关闭后，会退出循环
	for update := range podUpdates {
		pod := update.Options.Pod

		// Decide whether to start the pod. If the pod was terminated prior to the pod being allowed
		// to start, we have to clean it up and then exit the pod worker loop.
		// Pod不允许启动时直接退出函数，可以启动则等待至pod可以启动
		if !podStarted {
			canStart, canEverStart := p.allowPodStart(pod)
			if !canEverStart {
				p.completeUnstartedTerminated(pod)
				if start := update.Options.StartTime; !start.IsZero() {
					metrics.PodWorkerDuration.WithLabelValues("terminated").Observe(metrics.SinceInSeconds(start))
				}
				klog.V(4).InfoS("Processing pod event done", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", update.WorkType)
				return
			}
			if !canStart {
				klog.V(4).InfoS("Pod cannot start yet", "pod", klog.KObj(pod), "podUID", pod.UID)
				continue
			}
			podStarted = true
		}

		// 根据WorkType，调用sync三函数中的一个
		klog.V(4).InfoS("Processing pod event", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", update.WorkType)
		err := func() error {
			// The worker is responsible for ensuring the sync method sees the appropriate
			// status updates on resyncs (the result of the last sync), transitions to
			// terminating (no wait), or on terminated (whatever the most recent state is).
			// Only syncing and terminating can generate pod status changes, while terminated
			// pods ensure the most recent status makes it to the api server.
			var status *kubecontainer.PodStatus
			var err error
			switch {
			case update.Options.RunningPod != nil:
				// when we receive a running pod, we don't need status at all
			default:
				// wait until we see the next refresh from the PLEG via the cache (max 2s)
				// TODO: this adds ~1s of latency on all transitions from sync to terminating
				//  to terminated, and on all termination retries (including evictions). We should
				//  improve latency by making the the pleg continuous and by allowing pod status
				//  changes to be refreshed when key events happen (killPod, sync->terminating).
				//  Improving this latency also reduces the possibility that a terminated
				//  container's status is garbage collected before we have a chance to update the
				//  API server (thus losing the exit code).
				status, err = p.podCache.GetNewerThan(pod.UID, lastSyncTime)
			}
			if err != nil {
				// This is the legacy event thrown by manage pod loop all other events are now dispatched
				// from syncPodFn
				p.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedSync, "error determining status: %v", err)
				return err
			}

			ctx := p.contextForWorker(pod.UID)

			// Take the appropriate action (illegal phases are prevented by UpdatePod)
			switch {
			case update.WorkType == TerminatedPodWork:
				err = p.syncTerminatedPodFn(ctx, pod, status)

			case update.WorkType == TerminatingPodWork:
				var gracePeriod *int64
				if opt := update.Options.KillPodOptions; opt != nil {
					gracePeriod = opt.PodTerminationGracePeriodSecondsOverride
				}
				podStatusFn := p.acknowledgeTerminating(pod)

				err = p.syncTerminatingPodFn(ctx, pod, status, update.Options.RunningPod, gracePeriod, podStatusFn)

			default:
				err = p.syncPodFn(ctx, update.Options.UpdateType, pod, update.Options.MirrorPod, status)
			}

			lastSyncTime = time.Now()
			return err
		}()

		// 进行同步后处理
		switch {
		case err == context.Canceled:
			// when the context is cancelled we expect an update to already be queued
			klog.V(2).InfoS("Sync exited with context cancellation error", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", update.WorkType)

		case err != nil:
			// we will queue a retry
			klog.ErrorS(err, "Error syncing pod, skipping", "pod", klog.KObj(pod), "podUID", pod.UID)

		case update.WorkType == TerminatedPodWork:
			// we can shut down the worker
			p.completeTerminated(pod)
			if start := update.Options.StartTime; !start.IsZero() {
				metrics.PodWorkerDuration.WithLabelValues("terminated").Observe(metrics.SinceInSeconds(start))
			}
			klog.V(4).InfoS("Processing pod event done", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", update.WorkType)
			return

		case update.WorkType == TerminatingPodWork:
			// pods that don't exist in config don't need to be terminated, garbage collection will cover them
			if update.Options.RunningPod != nil {
				p.completeTerminatingRuntimePod(pod)
				if start := update.Options.StartTime; !start.IsZero() {
					metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
				}
				klog.V(4).InfoS("Processing pod event done", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", update.WorkType)
				return
			}
			// otherwise we move to the terminating phase
			// 
			p.completeTerminating(pod)
		}

		// queue a retry for errors if necessary, then put the next event in the channel if any
		p.completeWork(pod, err)
		if start := update.Options.StartTime; !start.IsZero() {
			metrics.PodWorkerDuration.WithLabelValues(update.Options.UpdateType.String()).Observe(metrics.SinceInSeconds(start))
		}
		klog.V(4).InfoS("Processing pod event done", "pod", klog.KObj(pod), "podUID", pod.UID, "updateType", update.WorkType)
	}
}
```