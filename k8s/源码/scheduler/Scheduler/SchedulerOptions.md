# SchedulerOptions
SchedulerOptions是Scheduler用于创建Scheduler结构体选项

## 结构体
```go
// pkg/scheduler/scheduler.go
type schedulerOptions struct {
	componentConfigVersion   string
	kubeConfig               *restclient.Config
	percentageOfNodesToScore int32
	podInitialBackoffSeconds int64
	podMaxBackoffSeconds     int64
	// Contains out-of-tree plugins to be merged with the in-tree registry.
	frameworkOutOfTreeRegistry frameworkruntime.Registry
	profiles                   []schedulerapi.KubeSchedulerProfile
	extenders                  []schedulerapi.Extender
	frameworkCapturer          FrameworkCapturer
	parallelism                int32
	applyDefaultProfile        bool
}
```
1. componentConfigVersion：KubeSchedulerConfiguration资源对象的api版本
2. percentageOfNodesToScore：需要进行打分操作的节点占节点数的百分比
3. podInitialBackoffSeconds，podMaxBackoffSeconds：指数等待时间参数
4. frameworkOutOfTreeRegistry：已经与内部Registry进行合并后的外部Registry
5. profiles：插件配置