# Configurator
持有创建Scheduler实例所需要的其他参数

## 结构体
```go
// pkg/scheduler/factory.go
// Configurator defines I/O, caching, and other functionality needed to
// construct a new scheduler.
type Configurator struct {
	client     clientset.Interface
	kubeConfig *restclient.Config

	recorderFactory profile.RecorderFactory

	informerFactory informers.SharedInformerFactory

	// Close this to stop all reflectors
	StopEverything <-chan struct{}

	schedulerCache internalcache.Cache

	componentConfigVersion string

	// Always check all predicates even if the middle of one predicate fails.
	alwaysCheckAllPredicates bool

	// percentageOfNodesToScore specifies percentage of all nodes to score in each scheduling cycle.
	percentageOfNodesToScore int32

	podInitialBackoffSeconds int64

	podMaxBackoffSeconds int64

	profiles          []schedulerapi.KubeSchedulerProfile
	registry          frameworkruntime.Registry
	nodeInfoSnapshot  *internalcache.Snapshot
	extenders         []schedulerapi.Extender
	frameworkCapturer FrameworkCapturer
	parallellism      int32
	// A "cluster event" -> "plugin names" map.
	clusterEventMap map[framework.ClusterEvent]sets.String
}
```

## create
create函数实例化一个Scheduler实例
```go
// pkg/scheduler/factory.go
// create a scheduler from a set of registered plugins.
func (c *Configurator) create() (*Scheduler, error) {
	var extenders []framework.Extender
	var ignoredExtendedResources []string
	// 实例化Extender
	if len(c.extenders) != 0 {
		var ignorableExtenders []framework.Extender
		for ii := range c.extenders {
			klog.V(2).InfoS("Creating extender", "extender", c.extenders[ii])
			extender, err := NewHTTPExtender(&c.extenders[ii])
			if err != nil {
				return nil, err
			}
			if !extender.IsIgnorable() {
				extenders = append(extenders, extender)
			} else {
				ignorableExtenders = append(ignorableExtenders, extender)
			}
			for _, r := range c.extenders[ii].ManagedResources {
				if r.IgnoredByScheduler {
					ignoredExtendedResources = append(ignoredExtendedResources, r.Name)
				}
			}
		}
		// place ignorable extenders to the tail of extenders
		extenders = append(extenders, ignorableExtenders...)
	}

	// If there are any extended resources found from the Extenders, append them to the pluginConfig for each profile.
	// This should only have an effect on ComponentConfig, where it is possible to configure Extenders and
	// plugin args (and in which case the extender ignored resources take precedence).
	// For earlier versions, using both policy and custom plugin config is disallowed, so this should be the only
	// plugin config for this plugin.
	if len(ignoredExtendedResources) > 0 {
		for i := range c.profiles {
			prof := &c.profiles[i]
			var found = false
			for k := range prof.PluginConfig {
				if prof.PluginConfig[k].Name == noderesources.FitName {
					// Update the existing args
					pc := &prof.PluginConfig[k]
					args, ok := pc.Args.(*schedulerapi.NodeResourcesFitArgs)
					if !ok {
						return nil, fmt.Errorf("want args to be of type NodeResourcesFitArgs, got %T", pc.Args)
					}
					args.IgnoredResources = ignoredExtendedResources
					found = true
					break
				}
			}
			if !found {
				return nil, fmt.Errorf("can't find NodeResourcesFitArgs in plugin config")
			}
		}
	}

	// The nominator will be passed all the way to framework instantiation.
	nominator := internalqueue.NewPodNominator(c.informerFactory.Core().V1().Pods().Lister())

	// 使用profile的NewMap来创建Framework
	profiles, err := profile.NewMap(c.profiles, c.registry, c.recorderFactory,
		frameworkruntime.WithComponentConfigVersion(c.componentConfigVersion),
		frameworkruntime.WithClientSet(c.client),
		frameworkruntime.WithKubeConfig(c.kubeConfig),
		frameworkruntime.WithInformerFactory(c.informerFactory),
		frameworkruntime.WithSnapshotSharedLister(c.nodeInfoSnapshot),
		frameworkruntime.WithRunAllFilters(c.alwaysCheckAllPredicates),
		frameworkruntime.WithPodNominator(nominator),
		frameworkruntime.WithCaptureProfile(frameworkruntime.CaptureProfile(c.frameworkCapturer)),
		frameworkruntime.WithClusterEventMap(c.clusterEventMap),
		frameworkruntime.WithParallelism(int(c.parallellism)),
		frameworkruntime.WithExtenders(extenders),
	)
	if err != nil {
		return nil, fmt.Errorf("initializing profiles: %v", err)
	}
	if len(profiles) == 0 {
		return nil, errors.New("at least one profile is required")
	}

	// 使用排序插件生成lessFn
	// Profiles are required to have equivalent queue sort plugins.
	lessFn := profiles[c.profiles[0].SchedulerName].QueueSortFunc()
	// 实例化SchedulingQueue
	podQueue := internalqueue.NewSchedulingQueue(
		lessFn,
		c.informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(c.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(c.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodNominator(nominator),
		internalqueue.WithClusterEventMap(c.clusterEventMap),
	)

	// Setup cache debugger.
	debugger := cachedebugger.New(
		c.informerFactory.Core().V1().Nodes().Lister(),
		c.informerFactory.Core().V1().Pods().Lister(),
		c.schedulerCache,
		podQueue,
	)
	debugger.ListenForSignal(c.StopEverything)

	// 将选取Node的流程封装
	algo := NewGenericScheduler(
		c.schedulerCache,
		c.nodeInfoSnapshot,
		c.percentageOfNodesToScore,
	)

	return &Scheduler{
		SchedulerCache:  c.schedulerCache,
		Algorithm:       algo,
		Extenders:       extenders,
		Profiles:        profiles,
		NextPod:         internalqueue.MakeNextPodFunc(podQueue),
		Error:           MakeDefaultErrorFunc(c.client, c.informerFactory.Core().V1().Pods().Lister(), podQueue, c.schedulerCache),
		StopEverything:  c.StopEverything,
		SchedulingQueue: podQueue,
	}, nil
}
```