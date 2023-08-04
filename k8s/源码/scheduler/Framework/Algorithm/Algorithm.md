# Algorithm
Algorithm定义了一个通过调度框架来调度Pod的接口，实现类为`genericScheduler`

## 接口
```go
// pkg/scheduler/generic_scheduler.go
type ScheduleAlgorithm interface {
	Schedule(context.Context, []framework.Extender, framework.Framework, *framework.CycleState, *v1.Pod) (scheduleResult ScheduleResult, err error)
}
```
1. Schedule：
   1. []framework.Extender：调度框架的扩展，一般是空
   2. framework.Framework：调度框架
   3. *framework.CycleState：循环状态
   4. *v1.Pod：pod对象