# FrameworkOptions
FrameworkOptions是用于构建Framework的一个结构体，保存了Framework需要使用的一些参数

## 结构体
```go
// pkg/scheduler/framework/runtime/framework.go
type frameworkOptions struct {
	componentConfigVersion string
	clientSet              clientset.Interface
	kubeConfig             *restclient.Config
	eventRecorder          events.EventRecorder
	informerFactory        informers.SharedInformerFactory
	snapshotSharedLister   framework.SharedLister
	metricsRecorder        *metricsRecorder
	podNominator           framework.PodNominator
	extenders              []framework.Extender
	runAllFilters          bool
	captureProfile         CaptureProfile
	clusterEventMap        map[framework.ClusterEvent]sets.String
	parallelizer           parallelize.Parallelizer
}
```

## 默认函数
默认的frameworkOptions，初始化了一些参数
```go
// pkg/scheduler/framework/runtime/framework.go
func defaultFrameworkOptions() frameworkOptions {
	return frameworkOptions{
        // 指标记录
		metricsRecorder: newMetricsRecorder(1000, time.Second),
        // 空map
		clusterEventMap: make(map[framework.ClusterEvent]sets.String),
        // 并行器，在进行打分等操作时，并行运行
		parallelizer:    parallelize.NewParallelizer(parallelize.DefaultParallelism),
	}
}
```

## 构造函数
frameworkOptions利用函数构造模式进行初始化
```go
// pkg/scheduler/framework/runtime/framework.go

// 用于初始化的函数签名
// Option for the frameworkImpl.
type Option func(*frameworkOptions)

// KubeSchedulerConfiguration资源对象的api版本
// WithComponentConfigVersion sets the component config version to the
// KubeSchedulerConfiguration version used. The string should be the full
// scheme group/version of the external type we converted from (for example
// "kubescheduler.config.k8s.io/v1beta2")
func WithComponentConfigVersion(componentConfigVersion string) Option {
	return func(o *frameworkOptions) {
		o.componentConfigVersion = componentConfigVersion
	}
}

// WithClientSet sets clientSet for the scheduling frameworkImpl.
func WithClientSet(clientSet clientset.Interface) Option {
	return func(o *frameworkOptions) {
		o.clientSet = clientSet
	}
}

// WithKubeConfig sets kubeConfig for the scheduling frameworkImpl.
func WithKubeConfig(kubeConfig *restclient.Config) Option {
	return func(o *frameworkOptions) {
		o.kubeConfig = kubeConfig
	}
}

// WithEventRecorder sets clientSet for the scheduling frameworkImpl.
func WithEventRecorder(recorder events.EventRecorder) Option {
	return func(o *frameworkOptions) {
		o.eventRecorder = recorder
	}
}

// WithInformerFactory sets informer factory for the scheduling frameworkImpl.
func WithInformerFactory(informerFactory informers.SharedInformerFactory) Option {
	return func(o *frameworkOptions) {
		o.informerFactory = informerFactory
	}
}

// WithSnapshotSharedLister sets the SharedLister of the snapshot.
func WithSnapshotSharedLister(snapshotSharedLister framework.SharedLister) Option {
	return func(o *frameworkOptions) {
		o.snapshotSharedLister = snapshotSharedLister
	}
}

// WithRunAllFilters sets the runAllFilters flag, which means RunFilterPlugins accumulates
// all failure Statuses.
func WithRunAllFilters(runAllFilters bool) Option {
	return func(o *frameworkOptions) {
		o.runAllFilters = runAllFilters
	}
}

// WithPodNominator sets podNominator for the scheduling frameworkImpl.
func WithPodNominator(nominator framework.PodNominator) Option {
	return func(o *frameworkOptions) {
		o.podNominator = nominator
	}
}

// WithExtenders sets extenders for the scheduling frameworkImpl.
func WithExtenders(extenders []framework.Extender) Option {
	return func(o *frameworkOptions) {
		o.extenders = extenders
	}
}

// WithParallelism sets parallelism for the scheduling frameworkImpl.
func WithParallelism(parallelism int) Option {
	return func(o *frameworkOptions) {
		o.parallelizer = parallelize.NewParallelizer(parallelism)
	}
}

// CaptureProfile is a callback to capture a finalized profile.
type CaptureProfile func(config.KubeSchedulerProfile)

// WithCaptureProfile sets a callback to capture the finalized profile.
func WithCaptureProfile(c CaptureProfile) Option {
	return func(o *frameworkOptions) {
		o.captureProfile = c
	}
}

// WithClusterEventMap sets clusterEventMap for the scheduling frameworkImpl.
func WithClusterEventMap(m map[framework.ClusterEvent]sets.String) Option {
	return func(o *frameworkOptions) {
		o.clusterEventMap = m
	}
}
```