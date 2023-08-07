# Scheduler
Scheduler是kube-scheduler的主体结构体，它会监控所有未被调度的Pod，寻找合适的节点，将Pod与节点进行绑定

## 结构体
```go
// pkg/scheduler/scheduler.go
// Scheduler watches for new unscheduled pods. It attempts to find
// nodes that they fit on and writes bindings back to the api server.
type Scheduler struct {
	// It is expected that changes made via SchedulerCache will be observed
	// by NodeLister and Algorithm.
	SchedulerCache internalcache.Cache

	Algorithm ScheduleAlgorithm

	Extenders []framework.Extender

	// NextPod should be a function that blocks until the next pod
	// is available. We don't use a channel for this, because scheduling
	// a pod may take some amount of time and we don't want pods to get
	// stale while they sit in a channel.
	NextPod func() *framework.QueuedPodInfo

	// Error is called if there is an error. It is passed the pod in
	// question, and the error
	Error func(*framework.QueuedPodInfo, error)

	// Close this to shut down the scheduler.
	StopEverything <-chan struct{}

	// SchedulingQueue holds pods to be scheduled
	SchedulingQueue internalqueue.SchedulingQueue

	// Profiles are the scheduling profiles.
	Profiles profile.Map

	client clientset.Interface
}
```
1. SchedulerCache：scheduler主要的缓存结构，缓存了所有的Node和带调度的Pod
2. Algorithm：scheduler主要的调度算法
3. Extenders：调度器外部扩展点，用于调度非k8s直接管理的资源
4. NextPod：一个用于获取下一个待调度Pod的函数，如果没有下一个等待调度的Pod，该函数会阻塞
5. Error：一个用于处理调度时错误的函数
6. StopEverything：接收停止信号的channel
7. SchedulingQueue：待调度的Pod队列
8. Profiles：schedulerName到schedulerFramework的映射。默认情况下只有一个名为default-scheduler的schedulerFramework
9. client：clientSet

## Run
启动Scheduler
```go
// pkg/scheduler/scheduler.go
func (sched *Scheduler) Run(ctx context.Context) {
	// 启动Queue
	sched.SchedulingQueue.Run()
	// 阻塞并一直执行schdulerOne方法知道context退出
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}
```

## 构造函数
通过若干Option函数，构造Scheduler
```go
// pkg/scheduler/scheduler.go
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

	// 默认的Option
	options := defaultSchedulerOptions
	// 对默认的Option进行修改
	for _, opt := range opts {
		opt(&options)
	}

	if options.applyDefaultProfile {
		var versionedCfg v1beta3.KubeSchedulerConfiguration
		scheme.Scheme.Default(&versionedCfg)
		cfg := config.KubeSchedulerConfiguration{}
		if err := scheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
			return nil, err
		}
		options.profiles = cfg.Profiles
	}
	schedulerCache := internalcache.New(durationToExpireAssumedPod, stopEverything)

	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	snapshot := internalcache.NewEmptySnapshot()
	clusterEventMap := make(map[framework.ClusterEvent]sets.String)

	configurator := &Configurator{
		componentConfigVersion:   options.componentConfigVersion,
		client:                   client,
		kubeConfig:               options.kubeConfig,
		recorderFactory:          recorderFactory,
		informerFactory:          informerFactory,
		schedulerCache:           schedulerCache,
		StopEverything:           stopEverything,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		podInitialBackoffSeconds: options.podInitialBackoffSeconds,
		podMaxBackoffSeconds:     options.podMaxBackoffSeconds,
		profiles:                 append([]schedulerapi.KubeSchedulerProfile(nil), options.profiles...),
		registry:                 registry,
		nodeInfoSnapshot:         snapshot,
		extenders:                options.extenders,
		frameworkCapturer:        options.frameworkCapturer,
		parallellism:             options.parallelism,
		clusterEventMap:          clusterEventMap,
	}

	metrics.Register()

	// Create the config from component config
	sched, err := configurator.create()
	if err != nil {
		return nil, fmt.Errorf("couldn't create scheduler: %v", err)
	}

	// Additional tweaks to the config produced by the configurator.
	sched.StopEverything = stopEverything
	sched.client = client

	addAllEventHandlers(sched, informerFactory, dynInformerFactory, unionedGVKs(clusterEventMap))

	return sched, nil
}
```