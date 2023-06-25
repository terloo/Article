# RuntimeService
RuntimeService定义了容器运行时Service接口规范，实现类是remoteRuntimeService

## 接口
```go
// vendor/k8s.io/cri-api/pkg/apis/services.go

// 容器运行时的组合接口
type RuntimeService interface {
	RuntimeVersioner
	ContainerManager
	PodSandboxManager
	ContainerStatsManager

	// UpdateRuntimeConfig updates runtime configuration if specified
    // 更新容器运行时配置
	UpdateRuntimeConfig(runtimeConfig *runtimeapi.RuntimeConfig) error
	// Status returns the status of the runtime.
    // 获取容器运行时状态
	Status() (*runtimeapi.RuntimeStatus, error)
}

// 获取容器运行时的版本信息
type RuntimeVersioner interface {
	// Version returns the runtime name, runtime version and runtime API version
	Version(apiVersion string) (*runtimeapi.VersionResponse, error)
}

// 容器管理器
type ContainerManager interface {
	// CreateContainer creates a new container in specified PodSandbox.
	CreateContainer(podSandboxID string, config *runtimeapi.ContainerConfig, sandboxConfig *runtimeapi.PodSandboxConfig) (string, error)
	// StartContainer starts the container.
	StartContainer(containerID string) error
	// StopContainer stops a running container with a grace period (i.e., timeout).
	StopContainer(containerID string, timeout int64) error
	// RemoveContainer removes the container.
	RemoveContainer(containerID string) error
	// ListContainers lists all containers by filters.
	ListContainers(filter *runtimeapi.ContainerFilter) ([]*runtimeapi.Container, error)
	// ContainerStatus returns the status of the container.
	ContainerStatus(containerID string) (*runtimeapi.ContainerStatus, error)
	// UpdateContainerResources updates the cgroup resources for the container.
	UpdateContainerResources(containerID string, resources *runtimeapi.LinuxContainerResources) error
	// ExecSync executes a command in the container, and returns the stdout output.
	// If command exits with a non-zero exit code, an error is returned.
	ExecSync(containerID string, cmd []string, timeout time.Duration) (stdout []byte, stderr []byte, err error)
	// Exec prepares a streaming endpoint to execute a command in the container, and returns the address.
	Exec(*runtimeapi.ExecRequest) (*runtimeapi.ExecResponse, error)
	// Attach prepares a streaming endpoint to attach to a running container, and returns the address.
	Attach(req *runtimeapi.AttachRequest) (*runtimeapi.AttachResponse, error)
	// ReopenContainerLog asks runtime to reopen the stdout/stderr log file
	// for the container. If it returns error, new container log file MUST NOT
	// be created.
	ReopenContainerLog(ContainerID string) error
}

// Pod沙箱管理器
type PodSandboxManager interface {
	// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
	// the sandbox is in ready state.
	RunPodSandbox(config *runtimeapi.PodSandboxConfig, runtimeHandler string) (string, error)
	// StopPodSandbox stops the sandbox. If there are any running containers in the
	// sandbox, they should be force terminated.
	StopPodSandbox(podSandboxID string) error
	// RemovePodSandbox removes the sandbox. If there are running containers in the
	// sandbox, they should be forcibly removed.
	RemovePodSandbox(podSandboxID string) error
	// PodSandboxStatus returns the Status of the PodSandbox.
	PodSandboxStatus(podSandboxID string) (*runtimeapi.PodSandboxStatus, error)
	// ListPodSandbox returns a list of Sandbox.
	ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error)
	// PortForward prepares a streaming endpoint to forward ports from a PodSandbox, and returns the address.
	PortForward(*runtimeapi.PortForwardRequest) (*runtimeapi.PortForwardResponse, error)
}

// 容器状态管理器
type ContainerStatsManager interface {
	// ContainerStats returns stats of the container. If the container does not
	// exist, the call returns an error.
	ContainerStats(containerID string) (*runtimeapi.ContainerStats, error)
	// ListContainerStats returns stats of all running containers.
	ListContainerStats(filter *runtimeapi.ContainerStatsFilter) ([]*runtimeapi.ContainerStats, error)
	// PodSandboxStats returns stats of the pod. If the pod does not
	// exist, the call returns an error.
	PodSandboxStats(podSandboxID string) (*runtimeapi.PodSandboxStats, error)
	// ListPodSandboxStats returns stats of all running pods.
	ListPodSandboxStats(filter *runtimeapi.PodSandboxStatsFilter) ([]*runtimeapi.PodSandboxStats, error)
}
```