# Run
消费DeploymentController的工作队列

## Run
传入工作协程数量，开始消费工作队列
```go
// pkg/controller/deployment/deployment_controller.go
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
	defer utilruntime.HandleCrash()
	defer dc.queue.ShutDown()

	klog.InfoS("Starting controller", "controller", "deployment")
	defer klog.InfoS("Shutting down controller", "controller", "deployment")

    // 等待deployment rs pod 三种informer初始同步完成
	if !cache.WaitForNamedCacheSync("deployment", ctx.Done(), dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
		return
	}

    // 根据传入的worker数，开启协程进行消费
	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, dc.worker, time.Second)
	}

	<-ctx.Done()
}
```

## syncHandler
实际的消费函数，从队列中消费出的key格式为`<namespace>/<resourceName>`
```go
// pkg/controller/deployment/deployment_controller.go
func (dc *DeploymentController) worker(ctx context.Context) {
	for dc.processNextWorkItem(ctx) {
	}
}

func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
    // 从队列中消费一个元素
	key, quit := dc.queue.Get()
	if quit {
		return false
	}
	defer dc.queue.Done(key)

    // 调用syncHnadler函数即syncDeployment方法
	err := dc.syncHandler(ctx, key.(string))
	dc.handleErr(err, key)

	return true
}
