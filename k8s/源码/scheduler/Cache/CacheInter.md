# Cache
Cache接口是Scheduler中用于存储调度相关信息的缓存，包括Node信息，待调度Pod信息等

## 接口
```go
type Cache interface {
	// NodeCount returns the number of nodes in the cache.
	// DO NOT use outside of tests.
	NodeCount() int

	// PodCount returns the number of pods in the cache (including those from deleted nodes).
	// DO NOT use outside of tests.
	PodCount() (int, error)

	// AssumePod assumes a pod scheduled and aggregates the pod's information into its node.
	// The implementation also decides the policy to expire pod before being confirmed (receiving Add event).
	// After expiration, its information would be subtracted.
	AssumePod(pod *v1.Pod) error

	// FinishBinding signals that cache for assumed pod can be expired
	FinishBinding(pod *v1.Pod) error

	// ForgetPod removes an assumed pod from cache.
	ForgetPod(pod *v1.Pod) error

	// AddPod either confirms a pod if it's assumed, or adds it back if it's expired.
	// If added back, the pod's information would be added again.
	AddPod(pod *v1.Pod) error

	// UpdatePod removes oldPod's information and adds newPod's information.
	UpdatePod(oldPod, newPod *v1.Pod) error

	// RemovePod removes a pod. The pod's information would be subtracted from assigned node.
	RemovePod(pod *v1.Pod) error

	// GetPod returns the pod from the cache with the same namespace and the
	// same name of the specified pod.
	GetPod(pod *v1.Pod) (*v1.Pod, error)

	// IsAssumedPod returns true if the pod is assumed and not expired.
	IsAssumedPod(pod *v1.Pod) (bool, error)

	// AddNode adds overall information about node.
	// It returns a clone of added NodeInfo object.
	AddNode(node *v1.Node) *framework.NodeInfo

	// UpdateNode updates overall information about node.
	// It returns a clone of updated NodeInfo object.
	UpdateNode(oldNode, newNode *v1.Node) *framework.NodeInfo

	// RemoveNode removes overall information about node.
	RemoveNode(node *v1.Node) error

	// UpdateSnapshot updates the passed infoSnapshot to the current contents of Cache.
	// The node info contains aggregated information of pods scheduled (including assumed to be)
	// on this node.
	// The snapshot only includes Nodes that are not deleted at the time this function is called.
	// nodeinfo.Node() is guaranteed to be not nil for all the nodes in the snapshot.
	UpdateSnapshot(nodeSnapshot *Snapshot) error

	// Dump produces a dump of the current cache.
	Dump() *Dump
}
```
1. AssumePod：假定一个Pod已经被调度，并将Pod信息聚合到节点信息中
2. FinishBinding：通知Cache一个assumedPod已经被绑定，可以过期
3. ForgetPod：通知Cache删除某个assumedPod
4. AddPod：添加一个Pod
5. UpdatePod：移除旧的Pod信息，添加新的Pod信息
6. RemovePod：移除一个Pod，这个Pod的信息将会被被分配的节点中删除
7. GetPod：获取一个Pod信息
8. IsAssumedPod：返回一个Pod是否被assume并且未过期
9. AddNode：添加一个Node信息
10. UpdateNode：更新一个Node信息
11. RemoveNode：移除一个Node信息

## 实现类
```go
type schedulerCache struct {
	stop   <-chan struct{}
	ttl    time.Duration
	period time.Duration

	// This mutex guards all fields within this cache struct.
	mu sync.RWMutex
	// a set of assumed pod keys.
	// The key could further be used to get an entry in podStates.
	assumedPods sets.String
	// a map from pod key to podState.
	podStates map[string]*podState
	nodes     map[string]*nodeInfoListItem
	// headNode points to the most recently updated NodeInfo in "nodes". It is the
	// head of the linked list.
	headNode *nodeInfoListItem
	nodeTree *nodeTree
	// A map from image name to its imageState.
	imageStates map[string]*imageState
}

type podState struct {
	pod *v1.Pod
	// Used by assumedPod to determinate expiration.
	deadline *time.Time
	// Used to block cache from expiring assumedPod if binding still runs
	bindingFinished bool
}

type imageState struct {
	// Size of the image
	size int64
	// A set of node names for nodes having this image present
	nodes sets.String
}

type nodeInfoListItem struct {
	info *framework.NodeInfo
	next *nodeInfoListItem
	prev *nodeInfoListItem
}
```
1. ttl：一个Pod被assume之后的过期时间
2. assumedPods：被assume的所有Pod
3. podStates：所有Pod信息的缓存
4. nodes：所有Node信息的缓存
5. headNode：最近一次的更新的NodeInfo，链表的头
6. nodeTree：节点数
7. imageStates：镜像名到镜像信息的映射

## 构造函数
```go
func New(ttl time.Duration, stop <-chan struct{}) Cache {
	cache := newSchedulerCache(ttl, cleanAssumedPeriod, stop)
	cache.run()
	return cache
}

func newSchedulerCache(ttl, period time.Duration, stop <-chan struct{}) *schedulerCache {
	return &schedulerCache{
		ttl:    ttl,
		period: period,
		stop:   stop,

		nodes:       make(map[string]*nodeInfoListItem),
		nodeTree:    newNodeTree(nil),
		assumedPods: make(sets.String),
		podStates:   make(map[string]*podState),
		imageStates: make(map[string]*imageState),
	}
}
```

## run
```go
// pkg/scheduler/internal/cache/cache.go
func (cache *schedulerCache) run() {
    // 定期清理过期的assumedPod
	go wait.Until(cache.cleanupExpiredAssumedPods, cache.period, cache.stop)
}

func (cache *schedulerCache) cleanupExpiredAssumedPods() {
	cache.cleanupAssumedPods(time.Now())
}
```