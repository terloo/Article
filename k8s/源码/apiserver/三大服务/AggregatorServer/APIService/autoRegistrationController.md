# autoRegisterController
autoRegisterController用于将APIService资源对象持久化到Etcd

## 接口
```go
// vendor/k8s.io/kube-aggergator/pkg/controllers/autoregister/autoregister_controller.go
type AutoAPIServiceRegistration interface {
	// 添加一个APIService，仅在服务启动时同步一次，用于k8s内置资源
	AddAPIServiceToSyncOnStart(in *v1.APIService)
	// 添加一个APIService，一直同步，用于CRD资源
	AddAPIServiceToSync(in *v1.APIService)
	// RemoveAPIServiceToSync removes an API service to auto-register.
	RemoveAPIServiceToSync(name string)
}
```

## 结构体
```go
// vendor/k8s.io/kube-aggregator/pkg/controllers/autoregister/autoregister_controller.go
type autoRegisterController struct {
	apiServiceLister listers.APIServiceLister
	apiServiceSynced cache.InformerSynced
	apiServiceClient apiregistrationclient.APIServicesGetter

	// apiServicesToSync同步锁
	apiServicesToSyncLock sync.RWMutex
	// 由于队列中只保存name，所以需要一个map保存APIService
	apiServicesToSync     map[string]*v1.APIService

	// 同步操作
	syncHandler func(apiServiceName string) error

	// track which services we have synced
	// syncedSuccessfully同步锁
	syncedSuccessfullyLock *sync.RWMutex
	// 保存哪些APIService已同步
	syncedSuccessfully     map[string]bool

	// remember names of services that existed when we started
	// 保存一个APIService是否开始处理
	apiServicesAtStart map[string]bool

	// queue is where incoming work is placed to de-dup and to allow "easy" rate limited requeues on errors
	queue workqueue.RateLimitingInterface
}
```

## 构造函数
```go
// vendor/k8s.io/kube-aggregator/pkg/controllers/autoregister/autoregister_controller.go
func NewAutoRegisterController(apiServiceInformer informers.APIServiceInformer, apiServiceClient apiregistrationclient.APIServicesGetter) *autoRegisterController {
	c := &autoRegisterController{
		apiServiceLister:  apiServiceInformer.Lister(),
		apiServiceSynced:  apiServiceInformer.Informer().HasSynced,
		apiServiceClient:  apiServiceClient,
		apiServicesToSync: map[string]*v1.APIService{},

		apiServicesAtStart: map[string]bool{},

		syncedSuccessfullyLock: &sync.RWMutex{},
		syncedSuccessfully:     map[string]bool{},

		// 限速队列
		queue: workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "autoregister"),
	}
	c.syncHandler = c.checkAPIService

	// 这个Informer在apiExtensionsServer
	apiServiceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			cast := obj.(*v1.APIService)
			// 向队列中添加的只有APIService的name
			c.queue.Add(cast.Name)
		},
		UpdateFunc: func(_, obj interface{}) {
			cast := obj.(*v1.APIService)
			c.queue.Add(cast.Name)
		},
		DeleteFunc: func(obj interface{}) {
			cast, ok := obj.(*v1.APIService)
			if !ok {
				tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
				if !ok {
					klog.V(2).Infof("Couldn't get object from tombstone %#v", obj)
					return
				}
				cast, ok = tombstone.Obj.(*v1.APIService)
				if !ok {
					klog.V(2).Infof("Tombstone contained unexpected object: %#v", obj)
					return
				}
			}
			c.queue.Add(cast.Name)
		},
	})

	return c
}
```

## 添加\移除APIService
添加APIService需要指定syncType，一般为manageOnStart(代表服务启动时生成)和manageContinuously(代表是CRD，需要不断的进行同步)
```go
// vendor/k8s.io/kube-aggregator/pkg/controllers/autoregister/autoregister_controller.go
const (
	// AutoRegisterManagedLabel is a label attached to the APIService that identifies how the APIService wants to be synced.
	AutoRegisterManagedLabel = "kube-aggregator.kubernetes.io/automanaged"

	// manageOnStart is a value for the AutoRegisterManagedLabel that indicates the APIService wants to be synced one time when the controller starts.
	manageOnStart = "onstart"
	// manageContinuously is a value for the AutoRegisterManagedLabel that indicates the APIService wants to be synced continuously.
	manageContinuously = "true"
)

func (c *autoRegisterController) AddAPIServiceToSyncOnStart(in *v1.APIService) {
	c.addAPIServiceToSync(in, manageOnStart)
}

// AddAPIServiceToSync registers an API service to sync continuously.
func (c *autoRegisterController) AddAPIServiceToSync(in *v1.APIService) {
	c.addAPIServiceToSync(in, manageContinuously)
}

func (c *autoRegisterController) addAPIServiceToSync(in *v1.APIService, syncType string) {
	c.apiServicesToSyncLock.Lock()
	defer c.apiServicesToSyncLock.Unlock()

	apiService := in.DeepCopy()
	if apiService.Labels == nil {
		apiService.Labels = map[string]string{}
	}
	// 根据syncType来修改apiService的自动合并标签
	apiService.Labels[AutoRegisterManagedLabel] = syncType

	c.apiServicesToSync[apiService.Name] = apiService
	// 加入到队列中让消费者进行处理
	c.queue.Add(apiService.Name)
}

// RemoveAPIServiceToSync deletes a registered APIService.
func (c *autoRegisterController) RemoveAPIServiceToSync(name string) {
	c.apiServicesToSyncLock.Lock()
	defer c.apiServicesToSyncLock.Unlock()

	delete(c.apiServicesToSync, name)
	c.queue.Add(name)
}
```

## 转换所有KubeAPIServer的API到KubeAPIServer
```go
// cmd/kube-apiserver/app/aggregator.go
// 传入的delegateAPIServer是KubeAPIServer的GenericAPIServer，返回内置API的所有APIService
func apiServicesToRegister(delegateAPIServer genericapiserver.DelegationTarget, registration autoregister.AutoAPIServiceRegistration) []*v1.APIService {
	apiServices := []*v1.APIService{}

	for _, curr := range delegateAPIServer.ListedPaths() {
		// 判断是否为corev1组
		if curr == "/api/v1" {
			apiService := makeAPIService(schema.GroupVersion{Group: "", Version: "v1"})
			// 添加APIService到autoRegisterController的队列中
			registration.AddAPIServiceToSyncOnStart(apiService)
			apiServices = append(apiServices, apiService)
			continue
		}

		// 判断是否为分组api，如果不是，跳过该路径
		if !strings.HasPrefix(curr, "/apis/") {
			continue
		}
		// this comes back in a list that looks like /apis/rbac.authorization.k8s.io/v1alpha1
		// 只接收GV的discovery路径
		tokens := strings.Split(curr, "/")
		if len(tokens) != 4 {
			continue
		}

		// 注册该分组API的APIService
		apiService := makeAPIService(schema.GroupVersion{Group: tokens[2], Version: tokens[3]})
		if apiService == nil {
			continue
		}
		registration.AddAPIServiceToSyncOnStart(apiService)
		apiServices = append(apiServices, apiService)
	}

	return apiServices
}

func makeAPIService(gv schema.GroupVersion) *v1.APIService {
	// apiVersionPriorities保存了所有内置API的组和版本优先级
	apiServicePriority, ok := apiVersionPriorities[gv]
	if !ok {
		// if we aren't found, then we shouldn't register ourselves because it could result in a CRD group version
		// being permanently stuck in the APIServices list.
		klog.Infof("Skipping APIService creation for %v", gv)
		return nil
	}
	return &v1.APIService{
		ObjectMeta: metav1.ObjectMeta{Name: gv.Version + "." + gv.Group},
		Spec: v1.APIServiceSpec{
			Group:                gv.Group,
			Version:              gv.Version,
			GroupPriorityMinimum: apiServicePriority.group,
			VersionPriority:      apiServicePriority.version,
		},
	}
}
```

## Run 消费者处理队列中的APIService
根据传入的worker数量，决定开启几个worker从队列中消费，最终调用的处理方法是checkAPIService
```go
func (c *autoRegisterController) Run(workers int, stopCh <-chan struct{}) {
	// don't let panics crash the process
	defer utilruntime.HandleCrash()
	// make sure the work queue is shutdown which will trigger workers to end
	defer c.queue.ShutDown()

	klog.Info("Starting autoregister controller")
	defer klog.Info("Shutting down autoregister controller")

	// wait for your secondary caches to fill before starting your work
	if !controllers.WaitForCacheSync("autoregister", stopCh, c.apiServiceSynced) {
		return
	}

	// record APIService objects that existed when we started
	if services, err := c.apiServiceLister.List(labels.Everything()); err == nil {
		for _, service := range services {
			c.apiServicesAtStart[service.Name] = true
		}
	}

	// start up your worker threads based on workers.  Some controllers have multiple kinds of workers
	for i := 0; i < workers; i++ {
		// runWorker will loop until "something bad" happens.  The .Until will then rekick the worker
		// after one second
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	// wait until we're told to stop
	<-stopCh
}

func (c *autoRegisterController) runWorker() {
	// hot loop until we're told to stop.  processNextWorkItem will automatically wait until there's work
	// available, so we don't worry about secondary waits
	for c.processNextWorkItem() {
	}
}

// processNextWorkItem deals with one key off the queue.  It returns false when it's time to quit.
func (c *autoRegisterController) processNextWorkItem() bool {
	// pull the next work item from queue.  It should be a key we use to lookup something in a cache
	key, quit := c.queue.Get()
	if quit {
		return false
	}
	// you always have to indicate to the queue that you've completed a piece of work
	defer c.queue.Done(key)

	// do your work on the key.  This method will contains your "do stuff" logic
	// 处理key的方法，其实是checkAPIService()
	err := c.syncHandler(key.(string))
	if err == nil {
		// if you had no error, tell the queue to stop tracking history for your key.  This will
		// reset things like failure counts for per-item rate limiting
		c.queue.Forget(key)
		return true
	}

	// there was a failure so be sure to report it.  This method allows for pluggable error handling
	// which can be used for things like cluster-monitoring
	utilruntime.HandleError(fmt.Errorf("%v failed with : %v", key, err))
	// since we failed, we should requeue the item to work on later.  This method will add a backoff
	// to avoid hotlooping on particular items (they're probably still not going to work right away)
	// and overall controller protection (everything I've done is broken, this controller needs to
	// calm down or it can starve other useful work) cases.
	c.queue.AddRateLimited(key)

	return true
}
```

## checkAPIService 处理函数
创建\删除\修改APIService时是通过回环客户端调用api-server
```go
// vendor/k8s.io/kube-aggregator/controllers/autoregister/autoregister_controller.go
//                                                 | A. desired: not found | B. desired: sync on start | C. desired: sync always
// ------------------------------------------------|-----------------------|---------------------------|------------------------
// 1. current: lookup error                        | error                 | error                     | error
// 2. current: not found                           | -                     | create once               | create
// 3. current: no sync                             | -                     | -                         | -
// 4. current: sync on start, not present at start | -                     | -                         | -
// 5. current: sync on start, present at start     | delete once           | update once               | update once
// 6. current: sync always                         | delete                | update once               | update
func (c *autoRegisterController) checkAPIService(name string) (err error) {
	// 从map中取出的apiService是desired的状态
	desired := c.GetAPIServiceToSync(name)
	// 从Lister(informers缓存)中取出的数据是current
	curr, err := c.apiServiceLister.Get(name)

	// if we've never synced this service successfully, record a successful sync.
	// 如果从未成功同步过，在成功同步后记录一下
	hasSynced := c.hasSyncedSuccessfully(name)
	if !hasSynced {
		defer func() {
			if err == nil {
				c.setSyncedSuccessfully(name)
			}
		}()
	}

	// 根据表格中的注释进行处理
	switch {
	// we had a real error, just return it (1A,1B,1C)
	case err != nil && !apierrors.IsNotFound(err):
		return err

	// we don't have an entry and we don't want one (2A)
	case apierrors.IsNotFound(err) && desired == nil:
		return nil

	// the local object only wants to sync on start and has already synced (2B,5B,6B "once" enforcement)
	case isAutomanagedOnStart(desired) && hasSynced:
		return nil

	// we don't have an entry and we do want one (2B,2C)
	case apierrors.IsNotFound(err) && desired != nil:
		// 通过回环客户端进行调用
		_, err := c.apiServiceClient.APIServices().Create(context.TODO(), desired, metav1.CreateOptions{})
		if apierrors.IsAlreadyExists(err) {
			// created in the meantime, we'll get called again
			return nil
		}
		return err

	// we aren't trying to manage this APIService (3A,3B,3C)
	case !isAutomanaged(curr):
		return nil

	// the remote object only wants to sync on start, but was added after we started (4A,4B,4C)
	case isAutomanagedOnStart(curr) && !c.apiServicesAtStart[name]:
		return nil

	// the remote object only wants to sync on start and has already synced (5A,5B,5C "once" enforcement)
	case isAutomanagedOnStart(curr) && hasSynced:
		return nil

	// we have a spurious APIService that we're managing, delete it (5A,6A)
	case desired == nil:
		opts := metav1.DeleteOptions{Preconditions: metav1.NewUIDPreconditions(string(curr.UID))}
		err := c.apiServiceClient.APIServices().Delete(context.TODO(), curr.Name, opts)
		if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
			// deleted or changed in the meantime, we'll get called again
			return nil
		}
		return err

	// if the specs already match, nothing for us to do
	case reflect.DeepEqual(curr.Spec, desired.Spec):
		return nil
	}

	// we have an entry and we have a desired, now we deconflict.  Only a few fields matter. (5B,5C,6B,6C)
	apiService := curr.DeepCopy()
	apiService.Spec = desired.Spec
	_, err = c.apiServiceClient.APIServices().Update(context.TODO(), apiService, metav1.UpdateOptions{})
	if apierrors.IsNotFound(err) || apierrors.IsConflict(err) {
		// deleted or changed in the meantime, we'll get called again
		return nil
	}
	return err
}
```