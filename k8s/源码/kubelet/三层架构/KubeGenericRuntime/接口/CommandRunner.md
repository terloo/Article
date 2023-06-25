# CommandRunner

## 接口
```go
type CommandRunner interface {
	// RunInContainer synchronously executes the command in the container, and returns the output.
	// If the command completes with a non-0 exit code, a k8s.io/utils/exec.ExitError will be returned.
	RunInContainer(id ContainerID, cmd []string, timeout time.Duration) ([]byte, error)
}
```