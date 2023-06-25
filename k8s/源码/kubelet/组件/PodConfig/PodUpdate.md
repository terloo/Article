# PodUpdate
PodUpdate定义了从来源发送过来的Pod操作

## 结构体
```go
// pkg/kubelet/types/pod_update.go
type PodUpdate struct {
	Pods   []*v1.Pod
	Op     PodOperation
	Source string
}

```
1. Pods：所有的Pod
2. Op：操作类型
3. Source：来源名

## 操作类型
```go
const (
	// SET is the current pod configuration.
	SET PodOperation = iota
	// ADD signifies pods that are new to this source.
	ADD
	// DELETE signifies pods that are gracefully deleted from this source.
	DELETE
	// REMOVE signifies pods that have been removed from this source.
	REMOVE
	// UPDATE signifies pods have been updated in this source.
	UPDATE
	// RECONCILE signifies pods that have unexpected status in this source,
	// kubelet should reconcile status with this source.
	RECONCILE
)
```
1. SET：该来源的全量数据
2. ADD：该Pod已添加
3. DELETE：该Pod已优雅删除
4. REMOVE：该Pod已删除
5. UPDATE：该Pod已更新
6. RECONCILE：该Pod的状态不正确，kubelet需要与源协调该Pod的状态