# StatusManager

## 作用
1. 缓存Pod的状态(corev1.PodStatus形式)
2. 在收到更新Pod状态(corev1.PodStatus形式)请求时，比较新旧状态。如果状态发生了改变，则将新状态同步到api-server；没改变则不进行操作。
3. 在同步新状态时，判断该Pod是否可以被移除(Pod被标记删除且资源回收完毕)，如果可以移除，则还应删除Pod
4. 如果需要同步状态是static pod，则同步其mirror pod
5. 定时遍历已缓存的所有pod，找出可以被移除的pod将其删除，如果缓存中状态与PodManager中状态不一致，则将缓存中状态同步到api-server触发一次协调状态的操作

## 接口
```go
// kubelet/status/status_manager.go
type PodStatusProvider interface {
	// GetPodStatus returns the cached status for the provided pod UID, as well as whether it
	// was a cache hit.
    // 通过pod的uid获取Status字段
	GetPodStatus(uid types.UID) (v1.PodStatus, bool)
}

type Manager interface {
	PodStatusProvider

	// Start the API server status sync loop.
	Start()

	// SetPodStatus caches updates the cached status for the given pod, and triggers a status update.
	SetPodStatus(pod *v1.Pod, status v1.PodStatus)

	// SetContainerReadiness updates the cached container status with the given readiness, and
	// triggers a status update.
	SetContainerReadiness(podUID types.UID, containerID kubecontainer.ContainerID, ready bool)

	// SetContainerStartup updates the cached container status with the given startup, and
	// triggers a status update.
	SetContainerStartup(podUID types.UID, containerID kubecontainer.ContainerID, started bool)

	// TerminatePod resets the container status for the provided pod to terminated and triggers
	// a status update.
	TerminatePod(pod *v1.Pod)

	// RemoveOrphanedStatuses scans the status cache and removes any entries for pods not included in
	// the provided podUIDs.
	RemoveOrphanedStatuses(podUIDs map[types.UID]bool)
}
```
1. Start：启动StatusManager
2. SetPodStatus：更新缓存并触发一次同步
3. SetContainerReadiness：设置Pod.Status.ContainerStatuses中某个container是否为ready状态，触发一次同步
4. SetContainerStartup：设置Pod.Status.ContainerStatuses中某个container是否为start状态，触发一次同步
5. TerminatePod：设置Pod.Status.ContainerStatuses所有Container状态为Terminated状态，触发一次同步
6. RemoveOrphanedStatuses：移除指定pid的缓存

## 实现类
```go
// kubelet/status/status_manager.go
// corev1.PodStatus的包装类，封装了一个版本号，用于丢弃版本号较低的versionedPodStatus
type versionedPodStatus struct {
	status v1.PodStatus
	// Monotonically increasing version number (per pod).
	version uint64
	// Pod name & namespace, for sending updates to API server.
	podName      string
	podNamespace string
}

type podStatusSyncRequest struct {
	podUID types.UID
	status versionedPodStatus
}

type manager struct {
	kubeClient clientset.Interface
	podManager kubepod.Manager
	// Map from pod UID to sync status of the corresponding pod.
	podStatuses      map[types.UID]versionedPodStatus
	podStatusesLock  sync.RWMutex
	podStatusChannel chan podStatusSyncRequest
	// Map from (mirror) pod UID to latest status version successfully sent to the API server.
	// apiStatusVersions must only be accessed from the sync thread.
	apiStatusVersions map[kubetypes.MirrorPodUID]uint64
	podDeletionSafety PodDeletionSafetyProvider
}
```
1. kubeClient：api-server客户端
2. podStatuses：uid到versionedPodStatus的映射
3. podStatusChannel：同步请求使用的channel
4. apiStatusVersions：已经同步到api-server的versionedPodStatus最新版本号，key是正常pod或者mirror pod的uid
5. podDeletionSafety：用于判断能否安全删除的类，实现类为kubelet

## 构造函数
```go
// kubelet/status/status_manager.go
func NewManager(kubeClient clientset.Interface, podManager kubepod.Manager, podDeletionSafety PodDeletionSafetyProvider) Manager {
	return &manager{
		kubeClient:        kubeClient,
		podManager:        podManager,
		podStatuses:       make(map[types.UID]versionedPodStatus),
		podStatusChannel:  make(chan podStatusSyncRequest, 1000), // Buffer up to 1000 statuses
		apiStatusVersions: make(map[kubetypes.MirrorPodUID]uint64),
		podDeletionSafety: podDeletionSafety,
	}
}
```

## Start
启动api-server同步循环
```go
// kubelet/status/status_manager.go
func (m *manager) Start() {
	// Don't start the status manager if we don't have a client. This will happen
	// on the master, where the kubelet is responsible for bootstrapping the pods
	// of the master components.
    // 如果kubeClient是nil，则不进行同步
	if m.kubeClient == nil {
		klog.InfoS("Kubernetes client is nil, not starting status manager")
		return
	}

	klog.InfoS("Starting to sync pod status with apiserver")

	//nolint:staticcheck // SA1015 Ticker can leak since this is only called once and doesn't handle termination.
	syncTicker := time.NewTicker(syncPeriod).C

	// syncPod and syncBatch share the same go routine to avoid sync races.
	go wait.Forever(func() {
		for {
			select {
			case syncRequest := <-m.podStatusChannel:
				klog.V(5).InfoS("Status Manager: syncing pod with status from podStatusChannel",
					"podUID", syncRequest.podUID,
					"statusVersion", syncRequest.status.version,
					"status", syncRequest.status.status)
					// podStatusChannel接收到消息时同步某个Pod
				m.syncPod(syncRequest.podUID, syncRequest.status)
			case <-syncTicker:
				klog.V(5).InfoS("Status Manager: syncing batch")
				// remove any entries in the status channel since the batch will handle them
				// 忽略掉所有等待同步的消息，直接同步所有Pod
				for i := len(m.podStatusChannel); i > 0; i-- {
					<-m.podStatusChannel
				}
				m.syncBatch()
			}
		}
	}, 0)
}
```


