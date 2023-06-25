# KubeGenericRuntime
KubeGenericRuntime是kubelet操作cri的顶层接口，由三个接口组成：
1. Runtime：Runtime继承了ImageService，用于操作容器运行时和镜像相关操作
2. StreamingRuntime：对Pod或容器进行流式操作
3. CommandRunner：向容器发送命令行命令

## 接口
```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go
type KubeGenericRuntime interface {
	kubecontainer.Runtime
	kubecontainer.StreamingRuntime
	kubecontainer.CommandRunner
}
```