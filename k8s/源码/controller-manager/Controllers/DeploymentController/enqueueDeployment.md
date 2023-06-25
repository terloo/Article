# enqueueDeployment
将Deployment资源对象放入队列，等待处理

## 
```go
// pkg/controller/deployment/deployment_controller.go
func (dc *DeploymentController) enqueue(deployment *apps.Deployment) {
	key, err := controller.KeyFunc(deployment)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", deployment, err))
		return
	}

	dc.queue.Add(key)
}
```