# NodeInfo
NodeInfo用于缓存Node在调度层面的数据，是对Informer的缓存内容的缓存。  
这样在进行调度时，就不用再次计算Node相关的数据。

## 结构体
```go
// pkg/scheduler/framework/types.go
type NodeInfo struct {
	// Overall node information.
	node *v1.Node

	// Pods running on the node.
	Pods []*PodInfo

	// The subset of pods with affinity.
	PodsWithAffinity []*PodInfo

	// The subset of pods with required anti-affinity.
	PodsWithRequiredAntiAffinity []*PodInfo

	// Ports allocated on the node.
	UsedPorts HostPortInfo

	// Total requested resources of all pods on this node. This includes assumed
	// pods, which scheduler has sent for binding, but may not be scheduled yet.
	Requested *Resource
	// Total requested resources of all pods on this node with a minimum value
	// applied to each container's CPU and memory requests. This does not reflect
	// the actual resource requests for this node, but is used to avoid scheduling
	// many zero-request pods onto one node.
	NonZeroRequested *Resource
	// We store allocatedResources (which is Node.Status.Allocatable.*) explicitly
	// as int64, to avoid conversions and accessing map.
	Allocatable *Resource

	// ImageStates holds the entry of an image if and only if this image is on the node. The entry can be used for
	// checking an image's existence and advanced usage (e.g., image locality scheduling policy) based on the image
	// state information.
	ImageStates map[string]*ImageStateSummary

	// PVCRefCounts contains a mapping of PVC names to the number of pods on the node using it.
	// Keys are in the format "namespace/name".
	PVCRefCounts map[string]int

	// Whenever NodeInfo changes, generation is bumped.
	// This is used to avoid cloning it if the object didn't change.
	Generation int64
}
```
1. Requested：该node上所有pod的Requested的总和
2. NonZeroRequested：该node上所有未设置Requested的pod总和