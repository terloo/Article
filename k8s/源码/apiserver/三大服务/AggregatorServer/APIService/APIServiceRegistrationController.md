# APIServiceRegistrationController
APIServiceRegistrationController Watch APIService的变化，将其通知给AggregatorServer

## 结构体
```go
// vendor/k8s.io/kube-aggregator/pkg/service/apiservice_controller.go
type APIServiceRegistrationController struct {
    // APIHandlerManager，实现类为AggregatorServer
	apiHandlerManager APIHandlerManager

	apiServiceLister listers.APIServiceLister
	apiServiceSynced cache.InformerSynced

	// 方便注入测试，将该结构体的一个方法以字段的形式封装
	syncFn func(key string) error

	queue workqueue.RateLimitingInterface
}

// 通知AggregatorServer处理APIService的接口
type APIHandlerManager interface {
	AddAPIService(apiService *v1.APIService) error
	RemoveAPIService(apiServiceName string)
}
```

## 构造函数
```go
// vendor/k8s.io/kube-aggregator/pkg/server/apiservice_controller.go
func NewAPIServiceRegistrationController(apiServiceInformer informers.APIServiceInformer, apiHandlerManager APIHandlerManager) *APIServiceRegistrationController {
	c := &APIServiceRegistrationController{
		apiHandlerManager: apiHandlerManager,
		apiServiceLister:  apiServiceInformer.Lister(),
		apiServiceSynced:  apiServiceInformer.Informer().HasSynced,
		queue:             workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "APIServiceRegistrationController"),
	}

	apiServiceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    c.addAPIService,
		UpdateFunc: c.updateAPIService,
		DeleteFunc: c.deleteAPIService,
	})

	c.syncFn = c.sync

	return c
}

func (c *APIServiceRegistrationController) addAPIService(obj interface{}) {
	castObj := obj.(*v1.APIService)
	klog.V(4).Infof("Adding %s", castObj.Name)
	c.enqueueInternal(castObj)
}

func (c *APIServiceRegistrationController) updateAPIService(obj, _ interface{}) {
	castObj := obj.(*v1.APIService)
	klog.V(4).Infof("Updating %s", castObj.Name)
	c.enqueueInternal(castObj)
}

func (c *APIServiceRegistrationController) deleteAPIService(obj interface{}) {
	castObj, ok := obj.(*v1.APIService)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			klog.Errorf("Couldn't get object from tombstone %#v", obj)
			return
		}
		castObj, ok = tombstone.Obj.(*v1.APIService)
		if !ok {
			klog.Errorf("Tombstone contained object that is not expected %#v", obj)
			return
		}
	}
	klog.V(4).Infof("Deleting %q", castObj.Name)
	c.enqueueInternal(castObj)
}
```

## Run
```go
// vendor/k8s.io/kube-aggregator/pkg/server/apiservice_controller.go
func (c *APIServiceRegistrationController) Run(stopCh <-chan struct{}, handlerSyncedCh chan<- struct{}) {
	defer utilruntime.HandleCrash()
	defer c.queue.ShutDown()

	klog.Info("Starting APIServiceRegistrationController")
	defer klog.Info("Shutting down APIServiceRegistrationController")

	if !controllers.WaitForCacheSync("APIServiceRegistrationController", stopCh, c.apiServiceSynced) {
		return
	}

	/// initially sync all APIServices to make sure the proxy handler is complete
    // 在启动Controller时将所有的APIService全通知给AggregatorServer
	if err := wait.PollImmediateUntil(time.Second, func() (bool, error) {
		services, err := c.apiServiceLister.List(labels.Everything())
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("failed to initially list APIServices: %v", err))
			return false, nil
		}
		for _, s := range services {
			if err := c.apiHandlerManager.AddAPIService(s); err != nil {
				utilruntime.HandleError(fmt.Errorf("failed to initially sync APIService %s: %v", s.Name, err))
				return false, nil
			}
		}
		return true, nil
	}, stopCh); err == wait.ErrWaitTimeout {
		utilruntime.HandleError(fmt.Errorf("timed out waiting for proxy handler to initialize"))
		return
	} else if err != nil {
		panic(fmt.Errorf("unexpected error: %v", err))
	}
	close(handlerSyncedCh)

	// only start one worker thread since its a slow moving API and the aggregation server adding bits
	// aren't threadsafe
    // 只开启一个worker
	go wait.Until(c.runWorker, time.Second, stopCh)

	<-stopCh
}

func (c *APIServiceRegistrationController) runWorker() {
	for c.processNextWorkItem() {
	}
}

// processNextWorkItem deals with one key off the queue.  It returns false when it's time to quit.
// 取出队列中的元素，交给sync处理
func (c *APIServiceRegistrationController) processNextWorkItem() bool {
	key, quit := c.queue.Get()
	if quit {
		return false
	}
	defer c.queue.Done(key)

	err := c.syncFn(key.(string))
	if err == nil {
		c.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("%v failed with : %v", key, err))
	c.queue.AddRateLimited(key)

	return true
}
```

## sync 处理队列中APIService
将队列中的
```go
// vendor/k8s.io/kube-aggregator/pkg/server/apiservice_controller.go
// 处理APIService，即通知给AggregatorServer
func (c *APIServiceRegistrationController) sync(key string) error {
	apiService, err := c.apiServiceLister.Get(key)
	if apierrors.IsNotFound(err) {
		c.apiHandlerManager.RemoveAPIService(key)
		return nil
	}
	if err != nil {
		return err
	}

	return c.apiHandlerManager.AddAPIService(apiService)
}
```