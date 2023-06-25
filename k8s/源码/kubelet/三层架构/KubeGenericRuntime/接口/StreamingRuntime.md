# StreamingRuntime
StreamingRuntime是构成KubeGenericRuntime接口的三个接口之一  
定义了exec/attach/port-forward操作

## 接口
```go
// pkg/kubelet/container/runtime
type StreamingRuntime interface {
	GetExec(id ContainerID, cmd []string, stdin, stdout, stderr, tty bool) (*url.URL, error)
	GetAttach(id ContainerID, stdin, stdout, stderr, tty bool) (*url.URL, error)
	GetPortForward(podName, podNamespace string, podUID types.UID, ports []int32) (*url.URL, error)
}
```