# FrameworkImpl
frameworkImpl是Framework的实现类
FrameworkImpl的构建需要三个组件的参与
1. Registry：保存了所有Plugin的数据结构
2. KubeSchedulerProfile：保存了Framework名及其需要的Plugin和Plugin配置的数据结构
3. frameworkOptions：保存了Framework需要的一些其他参数

## 实现类
```go
// pkg/scheduler/framework/framework.go
type frameworkImpl struct {
	registry             Registry
	snapshotSharedLister framework.SharedLister
	waitingPods          *waitingPodsMap
	scorePluginWeight    map[string]int
    // 扩展点字段
	queueSortPlugins     []framework.QueueSortPlugin
	preFilterPlugins     []framework.PreFilterPlugin
	filterPlugins        []framework.FilterPlugin
	postFilterPlugins    []framework.PostFilterPlugin
	preScorePlugins      []framework.PreScorePlugin
	scorePlugins         []framework.ScorePlugin
	reservePlugins       []framework.ReservePlugin
	preBindPlugins       []framework.PreBindPlugin
	bindPlugins          []framework.BindPlugin
	postBindPlugins      []framework.PostBindPlugin
	permitPlugins        []framework.PermitPlugin

	clientSet       clientset.Interface
	kubeConfig      *restclient.Config
	eventRecorder   events.EventRecorder
	informerFactory informers.SharedInformerFactory

	metricsRecorder *metricsRecorder
	profileName     string

	extenders []framework.Extender
	framework.PodNominator

	parallelizer parallelize.Parallelizer

	// Indicates that RunFilterPlugins should accumulate all failed statuses and not return
	// after the first failure.
	runAllFilters bool
}
```
1. registry

## 构造函数
通过FrameworkOptions来构造frameworkImpl实例
```go
// pkg/scheduler/framework/runtime/framework.go

func defaultFrameworkOptions() frameworkOptions {
	return frameworkOptions{
		metricsRecorder: newMetricsRecorder(1000, time.Second),
		clusterEventMap: make(map[framework.ClusterEvent]sets.String),
		parallelizer:    parallelize.NewParallelizer(parallelize.DefaultParallelism),
	}
}

// NewFramework initializes plugins given the configuration and the registry.
func NewFramework(r Registry, profile *config.KubeSchedulerProfile, opts ...Option) (framework.Framework, error) {
    // 默认options
	options := defaultFrameworkOptions()
	for _, opt := range opts {
		opt(&options)
	}

	f := &frameworkImpl{
		registry:             r,
		snapshotSharedLister: options.snapshotSharedLister,
		scorePluginWeight:    make(map[string]int),
		waitingPods:          newWaitingPodsMap(),
		clientSet:            options.clientSet,
		kubeConfig:           options.kubeConfig,
		eventRecorder:        options.eventRecorder,
		informerFactory:      options.informerFactory,
		metricsRecorder:      options.metricsRecorder,
		runAllFilters:        options.runAllFilters,
		extenders:            options.extenders,
		PodNominator:         options.podNominator,
		parallelizer:         options.parallelizer,
	}

	if profile == nil {
		return f, nil
	}

	f.profileName = profile.SchedulerName
	if profile.Plugins == nil {
		return f, nil
	}

	// get needed plugins from config
    // 获取所有Enable的Plugin
	pg := f.pluginsNeeded(profile.Plugins)

	pluginConfig := make(map[string]runtime.Object, len(profile.PluginConfig))
    // 获取所有Enable的Plugin的配置
	for i := range profile.PluginConfig {
		name := profile.PluginConfig[i].Name
		if _, ok := pluginConfig[name]; ok {
			return nil, fmt.Errorf("repeated config for plugin %s", name)
		}
		pluginConfig[name] = profile.PluginConfig[i].Args
	}
	outputProfile := config.KubeSchedulerProfile{
		SchedulerName: f.profileName,
		Plugins:       profile.Plugins,
		PluginConfig:  make([]config.PluginConfig, 0, len(pg)),
	}

	pluginsMap := make(map[string]framework.Plugin)
	for name, factory := range r {
        // 初始化所有Enable的插件
		// initialize only needed plugins.
		if _, ok := pg[name]; !ok {
			continue
		}

		args := pluginConfig[name]
		if args != nil {
			outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
				Name: name,
				Args: args,
			})
		}
        // 调用Plugin的构造函数来构造插件
		p, err := factory(args, f)
		if err != nil {
			return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
		}
		pluginsMap[name] = p

		// Update ClusterEventMap in place.
		fillEventToPluginMap(p, options.clusterEventMap)
	}

    // 给frameworkImpl的扩展点字段赋值
	// initialize plugins per individual extension points
	for _, e := range f.getExtensionPoints(profile.Plugins) {
		if err := updatePluginList(e.slicePtr, *e.plugins, pluginsMap); err != nil {
			return nil, err
		}
	}

    // 将profile的MultiPoint字段展开，给framework的扩展点字段赋值
	// initialize multiPoint plugins to their expanded extension points
	if len(profile.Plugins.MultiPoint.Enabled) > 0 {
		if err := f.expandMultiPointPlugins(profile, pluginsMap); err != nil {
			return nil, err
		}
	}

	if len(f.queueSortPlugins) != 1 {
		return nil, fmt.Errorf("one queue sort plugin required for profile with scheduler name %q", profile.SchedulerName)
	}

    // 给framework的插件权重字段赋值
	if err := getScoreWeights(f, pluginsMap, append(profile.Plugins.Score.Enabled, profile.Plugins.MultiPoint.Enabled...)); err != nil {
		return nil, err
	}

	// Verifying the score weights again since Plugin.Name() could return a different
	// value from the one used in the configuration.
    // 校验是否有权重为0的字段，如果有，返回error
	for _, scorePlugin := range f.scorePlugins {
		if f.scorePluginWeight[scorePlugin.Name()] == 0 {
			return nil, fmt.Errorf("score plugin %q is not configured with weight", scorePlugin.Name())
		}
	}

    // queueSortPlugins插件有且只能有一个
	if len(f.queueSortPlugins) == 0 {
		return nil, fmt.Errorf("no queue sort plugin is enabled")
	}
	if len(f.queueSortPlugins) > 1 {
		return nil, fmt.Errorf("only one queue sort plugin can be enabled")
	}
	if len(f.bindPlugins) == 0 {
		return nil, fmt.Errorf("at least one bind plugin is needed")
	}

    // 调用一下captureProfile函数，传入最终的profile配置
	if options.captureProfile != nil {
		if len(outputProfile.PluginConfig) != 0 {
			sort.Slice(outputProfile.PluginConfig, func(i, j int) bool {
				return outputProfile.PluginConfig[i].Name < outputProfile.PluginConfig[j].Name
			})
		} else {
			outputProfile.PluginConfig = nil
		}
		options.captureProfile(outputProfile)
	}

    // 实例化完毕
	return f, nil
}
```

## 排序
获取SchedulingQueue的排序方法，实际上是直接使用queueSortPlugin的Less函数
```go
// pkg/scheduler/framework/runtime/framework.go
// QueueSortFunc returns the function to sort pods in scheduling queue
func (f *frameworkImpl) QueueSortFunc() framework.LessFunc {
	if f == nil {
		// If frameworkImpl is nil, simply keep their order unchanged.
		// NOTE: this is primarily for tests.
		return func(_, _ *framework.QueuedPodInfo) bool { return false }
	}

	if len(f.queueSortPlugins) == 0 {
		panic("No QueueSort plugin is registered in the frameworkImpl.")
	}

	// Only one QueueSort plugin can be enabled.
	return f.queueSortPlugins[0].Less
}
```

## 预筛选
```go
// RunPreFilterPlugins runs the set of configured PreFilter plugins. It returns
// *Status and its code is set to non-success if any of the plugins returns
// anything but Success. If a non-success status is returned, then the scheduling
// cycle is aborted.
func (f *frameworkImpl) RunPreFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (status *framework.Status) {
	startTime := time.Now()
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(preFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
	}()
	for _, pl := range f.preFilterPlugins {
        // 依次执行插件的预筛选步骤
		status = f.runPreFilterPlugin(ctx, pl, state, pod)
		if !status.IsSuccess() {
			status.SetFailedPlugin(pl.Name())
			if status.IsUnschedulable() {
				return status
			}
			return framework.AsStatus(fmt.Errorf("running PreFilter plugin %q: %w", pl.Name(), status.AsError())).WithFailedPlugin(pl.Name())
		}
	}

	return nil
}

func (f *frameworkImpl) runPreFilterPlugin(ctx context.Context, pl framework.PreFilterPlugin, state *framework.CycleState, pod *v1.Pod) *framework.Status {
	if !state.ShouldRecordPluginMetrics() {
		return pl.PreFilter(ctx, state, pod)
	}
	startTime := time.Now()
	status := pl.PreFilter(ctx, state, pod)
	f.metricsRecorder.observePluginDurationAsync(preFilter, pl.Name(), status, metrics.SinceInSeconds(startTime))
	return status
}
```