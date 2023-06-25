# PodStatus
PodStatus是kubelet中保存了该节点上Pod相关所有容器的状态，`corev1.PodStatus`可以调用Kubelet的方法由PodStatus转化得到

## 结构体
```go
// pkg/kubelet/container/runtime.go
type PodStatus struct {
	// ID of the pod.
	ID types.UID
	// Name of the pod.
	Name string
	// Namespace of the pod.
	Namespace string
	// All IPs assigned to this pod
	IPs []string
	// Status of containers in the pod.
    // Pod中所有Container的状态
	ContainerStatuses []*Status
	// Status of the pod sandbox.
	// Only for kuberuntime now, other runtime may keep it nil.
    // Pod中sandbox的状态，cri中定义的结构体
	SandboxStatuses []*runtimeapi.PodSandboxStatus
}

// Container的状态
type Status struct {
	// ID of the container.
	ID ContainerID
	// Name of the container.
	Name string
	// Status of the container.
	State State
	// Creation time of the container.
	CreatedAt time.Time
	// Start time of the container.
	StartedAt time.Time
	// Finish time of the container.
	FinishedAt time.Time
	// Exit code of the container.
	ExitCode int
	// Name of the image, this also includes the tag of the image,
	// the expected form is "NAME:TAG".
	Image string
	// ID of the image.
	ImageID string
	// Hash of the container, used for comparison.
	Hash uint64
	// Number of times that the container has been restarted.
	RestartCount int
	// A string explains why container is in such a status.
	Reason string
	// Message written by the container before exiting (stored in
	// TerminationMessagePath).
	Message string
}

// 容器ID，包括容器运行时类型和容器id
type ContainerID struct {
	// The type of the container runtime. e.g. 'docker'.
	Type string
	// The identification of the container, this is comsumable by
	// the underlying container runtime. (Note that the container
	// runtime interface still takes the whole struct as input).
	ID string
}
```
