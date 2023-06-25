# main函数

```go
// cmd/kube-controller-manager/controller-manager.go
func main() {
	command := app.NewControllerManagerCommand()
	code := cli.Run(command)
	os.Exit(code)
}
```