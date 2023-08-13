# SchedulingQueue
SchedulingQueue是scheduler中用于保存待调度Pod的队列，由三个队列组成

## 接口
调度队列的接口，实现类似cache.FIFO和cache.Heap
```go
// pkg/shcduler/internal/queue/scheduling_queue.go
// SchedulingQueue is an interface for a queue to store pods waiting to be scheduled.
// The interface follows a pattern similar to cache.FIFO and cache.Heap and
// makes it easy to use those data structures as a SchedulingQueue.
type SchedulingQueue interface {
	framework.PodNominator
	Add(pod *v1.Pod) error
	// Activate moves the given pods to activeQ iff they're in unschedulableQ or backoffQ.
	// The passed-in pods are originally compiled from plugins that want to activate Pods,
	// by injecting the pods through a reserved CycleState struct (PodsToActivate).
	Activate(pods map[string]*v1.Pod)
	// AddUnschedulableIfNotPresent adds an unschedulable pod back to scheduling queue.
	// The podSchedulingCycle represents the current scheduling cycle number which can be
	// returned by calling SchedulingCycle().
	AddUnschedulableIfNotPresent(pod *framework.QueuedPodInfo, podSchedulingCycle int64) error
	// SchedulingCycle returns the current number of scheduling cycle which is
	// cached by scheduling queue. Normally, incrementing this number whenever
	// a pod is popped (e.g. called Pop()) is enough.
	SchedulingCycle() int64
	// Pop removes the head of the queue and returns it. It blocks if the
	// queue is empty and waits until a new item is added to the queue.
	Pop() (*framework.QueuedPodInfo, error)
	Update(oldPod, newPod *v1.Pod) error
	Delete(pod *v1.Pod) error
	MoveAllToActiveOrBackoffQueue(event framework.ClusterEvent, preCheck PreEnqueueCheck)
	AssignedPodAdded(pod *v1.Pod)
	AssignedPodUpdated(pod *v1.Pod)
	PendingPods() []*v1.Pod
	// Close closes the SchedulingQueue so that the goroutine which is
	// waiting to pop items can exit gracefully.
	Close()
	// Run starts the goroutines managing the queue.
	Run()
}

type PodNominator interface {
	// AddNominatedPod adds the given pod to the nominator or
	// updates it if it already exists.
	AddNominatedPod(pod *PodInfo, nominatingInfo *NominatingInfo)
	// DeleteNominatedPodIfExists deletes nominatedPod from internal cache. It's a no-op if it doesn't exist.
	DeleteNominatedPodIfExists(pod *v1.Pod)
	// UpdateNominatedPod updates the <oldPod> with <newPod>.
	UpdateNominatedPod(oldPod *v1.Pod, newPodInfo *PodInfo)
	// NominatedPodsForNode returns nominatedPods on the given node.
	NominatedPodsForNode(nodeName string) []*PodInfo
}
```
1. Add：添加一个Pod到队列中
2. Activate：添加一个Pod到activeQ中
3. AddUnschedulableIfNotPresent：将一个在unschedulablePods队列中的Pod添加到activeQ中
4. SchedulingCycle：返回该队列中所有待调度的Pod
5. Pop：获取一个Pod
6. Update：更新一个Pod
7. Delete：删除一个Pod
8. MoveAllToActiveOrBackoffQueue：移动Pod到activeQ或者BackoffQ中

## 实现类
SchedulingQueue的实现是一个优先级队列，队列的头部是优先级最好的Pending状态的Pod。PriorityQueue由三个子队列组成：
1. activeQ：保存所有准备好被调度的Pod
2. unschedulableQ：保存所有经过一次调度流程但无法被成功调度的Pod
3. backoffQ：
```go
// pkg/scheduler/internal/queue/scheduling_queue.go
// PriorityQueue implements a scheduling queue.
// The head of PriorityQueue is the highest priority pending pod. This structure
// has three sub queues. One sub-queue holds pods that are being considered for
// scheduling. This is called activeQ and is a Heap. Another queue holds
// pods that are already tried and are determined to be unschedulable. The latter
// is called unschedulableQ. The third queue holds pods that are moved from
// unschedulable queues and will be moved to active queue when backoff are completed.
type PriorityQueue struct {
	// PodNominator abstracts the operations to maintain nominated Pods.
	framework.PodNominator

	stop  chan struct{}
	clock util.Clock
	// pod initial backoff duration.
	podInitialBackoffDuration time.Duration
	// pod maximum backoff duration.
	podMaxBackoffDuration time.Duration

	lock sync.RWMutex
	cond sync.Cond

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	activeQ *heap.Heap
	// podBackoffQ is a heap ordered by backoff expiry. Pods which have completed backoff
	// are popped from this heap before the scheduler looks at activeQ
	podBackoffQ *heap.Heap
	// unschedulableQ holds pods that have been tried and determined unschedulable.
	unschedulableQ *UnschedulablePodsMap
	// schedulingCycle represents sequence number of scheduling cycle and is incremented
	// when a pod is popped.
	schedulingCycle int64
	// moveRequestCycle caches the sequence number of scheduling cycle when we
	// received a move request. Unschedulable pods in and before this scheduling
	// cycle will be put back to activeQueue if we were trying to schedule them
	// when we received move request.
	moveRequestCycle int64

	clusterEventMap map[framework.ClusterEvent]sets.String

	// closed indicates that the queue is closed.
	// It is mainly used to let Pop() exit its control loop while waiting for an item.
	closed bool

	nsLister listersv1.NamespaceLister
}
```
1. framework.PodNominator：用于暂存抢占成功的Pod
2. podInitialBackoffDuration：backoff初始时间
3. podMaxBackoffDuration：backoff最大时间
4. schedulingCycle：表示调度周期数，每次Pod从队列中被Pop出来后，这个值就会+1