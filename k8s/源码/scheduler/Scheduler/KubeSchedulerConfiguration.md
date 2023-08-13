# KubeSchedulerConfiguration
KubeSchedulerConfiguration是kube-scheduler的配置资源对象

## 结构体
```go
// k8s.io/kube-scheduler/config/v1beta3/types.go
type KubeSchedulerConfiguration struct {
	metav1.TypeMeta `json:",inline"`

	// Parallelism defines the amount of parallelism in algorithms for scheduling a Pods. Must be greater than 0. Defaults to 16
	Parallelism *int32 `json:"parallelism,omitempty"`

	// LeaderElection defines the configuration of leader election client.
	LeaderElection componentbaseconfigv1alpha1.LeaderElectionConfiguration `json:"leaderElection"`

	// ClientConnection specifies the kubeconfig file and client connection
	// settings for the proxy server to use when communicating with the apiserver.
	ClientConnection componentbaseconfigv1alpha1.ClientConnectionConfiguration `json:"clientConnection"`

	// DebuggingConfiguration holds configuration for Debugging related features
	// TODO: We might wanna make this a substruct like Debugging componentbaseconfigv1alpha1.DebuggingConfiguration
	componentbaseconfigv1alpha1.DebuggingConfiguration `json:",inline"`

	// PercentageOfNodesToScore is the percentage of all nodes that once found feasible
	// for running a pod, the scheduler stops its search for more feasible nodes in
	// the cluster. This helps improve scheduler's performance. Scheduler always tries to find
	// at least "minFeasibleNodesToFind" feasible nodes no matter what the value of this flag is.
	// Example: if the cluster size is 500 nodes and the value of this flag is 30,
	// then scheduler stops finding further feasible nodes once it finds 150 feasible ones.
	// When the value is 0, default percentage (5%--50% based on the size of the cluster) of the
	// nodes will be scored.
	PercentageOfNodesToScore *int32 `json:"percentageOfNodesToScore,omitempty"`

	// PodInitialBackoffSeconds is the initial backoff for unschedulable pods.
	// If specified, it must be greater than 0. If this value is null, the default value (1s)
	// will be used.
	PodInitialBackoffSeconds *int64 `json:"podInitialBackoffSeconds,omitempty"`

	// PodMaxBackoffSeconds is the max backoff for unschedulable pods.
	// If specified, it must be greater than podInitialBackoffSeconds. If this value is null,
	// the default value (10s) will be used.
	PodMaxBackoffSeconds *int64 `json:"podMaxBackoffSeconds,omitempty"`

	// Profiles are scheduling profiles that kube-scheduler supports. Pods can
	// choose to be scheduled under a particular profile by setting its associated
	// scheduler name. Pods that don't specify any scheduler name are scheduled
	// with the "default-scheduler" profile, if present here.
	// +listType=map
	// +listMapKey=schedulerName
    // 对不同的调度框架的插件进行配置
	Profiles []KubeSchedulerProfile `json:"profiles,omitempty"`

	// Extenders are the list of scheduler extenders, each holding the values of how to communicate
	// with the extender. These extenders are shared by all scheduler profiles.
	// +listType=set
	Extenders []Extender `json:"extenders,omitempty"`
}

type KubeSchedulerProfile struct {
	// SchedulerName is the name of the scheduler associated to this profile.
	// If SchedulerName matches with the pod's "spec.schedulerName", then the pod
	// is scheduled with this profile.
    // 调度框架名
	SchedulerName *string `json:"schedulerName,omitempty"`

	// Plugins specify the set of plugins that should be enabled or disabled.
	// Enabled plugins are the ones that should be enabled in addition to the
	// default plugins. Disabled plugins are any of the default plugins that
	// should be disabled.
	// When no enabled or disabled plugin is specified for an extension point,
	// default plugins for that extension point will be used if there is any.
	// If a QueueSort plugin is specified, the same QueueSort Plugin and
	// PluginConfig must be specified for all profiles.
    // 调度框架启用或者禁用插件的配置
	Plugins *Plugins `json:"plugins,omitempty"`

	// PluginConfig is an optional set of custom plugin arguments for each plugin.
	// Omitting config args for a plugin is equivalent to using the default config
	// for that plugin.
	// +listType=map
	// +listMapKey=name
    // 插件内部配置项
	PluginConfig []PluginConfig `json:"pluginConfig,omitempty"`
}

// 配置插件在某扩展点是否启用/禁用，如果插件名指定为*则代表启用/禁用该扩展点
// Plugins include multiple extension points. When specified, the list of plugins for
// a particular extension point are the only ones enabled. If an extension point is
// omitted from the config, then the default set of plugins is used for that extension point.
// Enabled plugins are called in the order specified here, after default plugins. If they need to
// be invoked before default plugins, default plugins must be disabled and re-enabled here in desired order.
type Plugins struct {
	// QueueSort is a list of plugins that should be invoked when sorting pods in the scheduling queue.
	QueueSort PluginSet `json:"queueSort,omitempty"`

	// PreFilter is a list of plugins that should be invoked at "PreFilter" extension point of the scheduling framework.
	PreFilter PluginSet `json:"preFilter,omitempty"`

	// Filter is a list of plugins that should be invoked when filtering out nodes that cannot run the Pod.
	Filter PluginSet `json:"filter,omitempty"`

	// PostFilter is a list of plugins that are invoked after filtering phase, but only when no feasible nodes were found for the pod.
	PostFilter PluginSet `json:"postFilter,omitempty"`

	// PreScore is a list of plugins that are invoked before scoring.
	PreScore PluginSet `json:"preScore,omitempty"`

	// Score is a list of plugins that should be invoked when ranking nodes that have passed the filtering phase.
	Score PluginSet `json:"score,omitempty"`

	// Reserve is a list of plugins invoked when reserving/unreserving resources
	// after a node is assigned to run the pod.
	Reserve PluginSet `json:"reserve,omitempty"`

	// Permit is a list of plugins that control binding of a Pod. These plugins can prevent or delay binding of a Pod.
	Permit PluginSet `json:"permit,omitempty"`

	// PreBind is a list of plugins that should be invoked before a pod is bound.
	PreBind PluginSet `json:"preBind,omitempty"`

	// Bind is a list of plugins that should be invoked at "Bind" extension point of the scheduling framework.
	// The scheduler call these plugins in order. Scheduler skips the rest of these plugins as soon as one returns success.
	Bind PluginSet `json:"bind,omitempty"`

	// PostBind is a list of plugins that should be invoked after a pod is successfully bound.
	PostBind PluginSet `json:"postBind,omitempty"`

    // 配置插件在其所有实现的扩展点上是否启用/禁用，如果插件名字为*，则代表启用/禁用所有插件
	// MultiPoint is a simplified config section to enable plugins for all valid extension points.
	// Plugins enabled through MultiPoint will automatically register for every individual extension
	// point the plugin has implemented. Disabling a plugin through MultiPoint disables that behavior.
	// The same is true for disabling "*" through MultiPoint (no default plugins will be automatically registered).
	// Plugins can still be disabled through their individual extension points.
	//
	// In terms of precedence, plugin config follows this basic hierarchy
	//   1. Specific extension points
	//   2. Explicitly configured MultiPoint plugins
	//   3. The set of default plugins, as MultiPoint plugins
	// This implies that a higher precedence plugin will run first and overwrite any settings within MultiPoint.
	// Explicitly user-configured plugins also take a higher precedence over default plugins.
	// Within this hierarchy, an Enabled setting takes precedence over Disabled. For example, if a plugin is
	// set in both `multiPoint.Enabled` and `multiPoint.Disabled`, the plugin will be enabled. Similarly,
	// including `multiPoint.Disabled = '*'` and `multiPoint.Enabled = pluginA` will still register that specific
	// plugin through MultiPoint. This follows the same behavior as all other extension point configurations.
	MultiPoint PluginSet `json:"multiPoint,omitempty"`
}

// PluginSet specifies enabled and disabled plugins for an extension point.
// If an array is empty, missing, or nil, default plugins at that extension point will be used.
type PluginSet struct {
	// Enabled specifies plugins that should be enabled in addition to default plugins.
	// If the default plugin is also configured in the scheduler config file, the weight of plugin will
	// be overridden accordingly.
	// These are called after default plugins and in the same order specified here.
	// +listType=atomic
	Enabled []Plugin `json:"enabled,omitempty"`
	// Disabled specifies default plugins that should be disabled.
	// When all default plugins need to be disabled, an array containing only one "*" should be provided.
	// +listType=map
	// +listMapKey=name
	Disabled []Plugin `json:"disabled,omitempty"`
}

// Plugin specifies a plugin name and its weight when applicable. Weight is used only for Score plugins.
type Plugin struct {
	// Name defines the name of plugin
	Name string `json:"name"`
    // 权重，只对打分插件起作用
	// Weight defines the weight of plugin, only used for Score plugins.
	Weight *int32 `json:"weight,omitempty"`
}

// 通过一个资源对象来对插件本身进行配置
// PluginConfig specifies arguments that should be passed to a plugin at the time of initialization.
// A plugin that is invoked at multiple extension points is initialized once. Args can have arbitrary structure.
// It is up to the plugin to process these Args.
type PluginConfig struct {
	// Name defines the name of plugin being configured
	Name string `json:"name"`
	// Args defines the arguments passed to the plugins at the time of initialization. Args can have arbitrary structure.
	Args runtime.RawExtension `json:"args,omitempty"`
}

// 外部调度扩展
// Extender holds the parameters used to communicate with the extender. If a verb is unspecified/empty,
// it is assumed that the extender chose not to provide that extension.
type Extender struct {
	// URLPrefix at which the extender is available
	URLPrefix string
	// Verb for the filter call, empty if not supported. This verb is appended to the URLPrefix when issuing the filter call to extender.
	FilterVerb string
	// Verb for the preempt call, empty if not supported. This verb is appended to the URLPrefix when issuing the preempt call to extender.
	PreemptVerb string
	// Verb for the prioritize call, empty if not supported. This verb is appended to the URLPrefix when issuing the prioritize call to extender.
	PrioritizeVerb string
	// The numeric multiplier for the node scores that the prioritize call generates.
	// The weight should be a positive integer
	Weight int64
	// Verb for the bind call, empty if not supported. This verb is appended to the URLPrefix when issuing the bind call to extender.
	// If this method is implemented by the extender, it is the extender's responsibility to bind the pod to apiserver. Only one extender
	// can implement this function.
	BindVerb string
	// EnableHTTPS specifies whether https should be used to communicate with the extender
	EnableHTTPS bool
	// TLSConfig specifies the transport layer security config
	TLSConfig *ExtenderTLSConfig
	// HTTPTimeout specifies the timeout duration for a call to the extender. Filter timeout fails the scheduling of the pod. Prioritize
	// timeout is ignored, k8s/other extenders priorities are used to select the node.
	HTTPTimeout metav1.Duration
	// NodeCacheCapable specifies that the extender is capable of caching node information,
	// so the scheduler should only send minimal information about the eligible nodes
	// assuming that the extender already cached full details of all nodes in the cluster
	NodeCacheCapable bool
	// ManagedResources is a list of extended resources that are managed by
	// this extender.
	// - A pod will be sent to the extender on the Filter, Prioritize and Bind
	//   (if the extender is the binder) phases iff the pod requests at least
	//   one of the extended resources in this list. If empty or unspecified,
	//   all pods will be sent to this extender.
	// - If IgnoredByScheduler is set to true for a resource, kube-scheduler
	//   will skip checking the resource in predicates.
	// +optional
	ManagedResources []ExtenderManagedResource
	// Ignorable specifies if the extender is ignorable, i.e. scheduling should not
	// fail when the extender returns an error or is not reachable.
	Ignorable bool
}
```

## 默认配置
```go
// pkg/scheduler/apis/latest/latest.go
// Default creates a default configuration of the latest versioned type.
// This function needs to be updated whenever we bump the scheduler's component config version.
func Default() (*config.KubeSchedulerConfiguration, error) {
	versionedCfg := v1beta3.KubeSchedulerConfiguration{}
	versionedCfg.DebuggingConfiguration = *v1alpha1.NewRecommendedDebuggingConfiguration()

	// 使用scheme将其填充默认值，并转为内部版本结构
	scheme.Scheme.Default(&versionedCfg)
	cfg := config.KubeSchedulerConfiguration{}
	if err := scheme.Scheme.Convert(&versionedCfg, &cfg, nil); err != nil {
		return nil, err
	}
	// We don't set this field in pkg/scheduler/apis/config/{version}/conversion.go
	// because the field will be cleared later by API machinery during
	// conversion. See KubeSchedulerConfiguration internal type definition for
	// more details.
	cfg.TypeMeta.APIVersion = v1beta3.SchemeGroupVersion.String()
	return &cfg, nil
}

// Default函数
func SetDefaults_KubeSchedulerConfiguration(obj *v1beta3.KubeSchedulerConfiguration) {
	if obj.Parallelism == nil {
		obj.Parallelism = pointer.Int32Ptr(16)
	}

	// 添加一个空结构
	if len(obj.Profiles) == 0 {
		obj.Profiles = append(obj.Profiles, v1beta3.KubeSchedulerProfile{})
	}
	// Only apply a default scheduler name when there is a single profile.
	// Validation will ensure that every profile has a non-empty unique name.
	// 如果只有一个Profile，则将其命名为default-scheduler
	if len(obj.Profiles) == 1 && obj.Profiles[0].SchedulerName == nil {
		obj.Profiles[0].SchedulerName = pointer.StringPtr(v1.DefaultSchedulerName)
	}

	// Add the default set of plugins and apply the configuration.
	for i := range obj.Profiles {
		prof := &obj.Profiles[i]
		setDefaults_KubeSchedulerProfile(prof)
	}

	if obj.PercentageOfNodesToScore == nil {
		percentageOfNodesToScore := int32(config.DefaultPercentageOfNodesToScore)
		obj.PercentageOfNodesToScore = &percentageOfNodesToScore
	}

	if len(obj.LeaderElection.ResourceLock) == 0 {
		// Use lease-based leader election to reduce cost.
		// We migrated for EndpointsLease lock in 1.17 and starting in 1.20 we
		// migrated to Lease lock.
		obj.LeaderElection.ResourceLock = "leases"
	}
	if len(obj.LeaderElection.ResourceNamespace) == 0 {
		obj.LeaderElection.ResourceNamespace = v1beta3.SchedulerDefaultLockObjectNamespace
	}
	if len(obj.LeaderElection.ResourceName) == 0 {
		obj.LeaderElection.ResourceName = v1beta3.SchedulerDefaultLockObjectName
	}

	if len(obj.ClientConnection.ContentType) == 0 {
		obj.ClientConnection.ContentType = "application/vnd.kubernetes.protobuf"
	}
	// Scheduler has an opinion about QPS/Burst, setting specific defaults for itself, instead of generic settings.
	if obj.ClientConnection.QPS == 0.0 {
		obj.ClientConnection.QPS = 50.0
	}
	if obj.ClientConnection.Burst == 0 {
		obj.ClientConnection.Burst = 100
	}

	// Use the default LeaderElectionConfiguration options
	componentbaseconfigv1alpha1.RecommendedDefaultLeaderElectionConfiguration(&obj.LeaderElection)

	if obj.PodInitialBackoffSeconds == nil {
		val := int64(1)
		obj.PodInitialBackoffSeconds = &val
	}

	if obj.PodMaxBackoffSeconds == nil {
		val := int64(10)
		obj.PodMaxBackoffSeconds = &val
	}

	// Enable profiling by default in the scheduler
	if obj.EnableProfiling == nil {
		enableProfiling := true
		obj.EnableProfiling = &enableProfiling
	}

	// Enable contention profiling by default if profiling is enabled
	if *obj.EnableProfiling && obj.EnableContentionProfiling == nil {
		enableContentionProfiling := true
		obj.EnableContentionProfiling = &enableContentionProfiling
	}
}

// 给profile合并内置Plugin
func setDefaults_KubeSchedulerProfile(prof *v1beta3.KubeSchedulerProfile) {
	// Set default plugins.
	prof.Plugins = mergePlugins(getDefaultPlugins(), prof.Plugins)
	// Set default plugin configs.
	scheme := GetPluginArgConversionScheme()
	existingConfigs := sets.NewString()
	for j := range prof.PluginConfig {
		existingConfigs.Insert(prof.PluginConfig[j].Name)
		args := prof.PluginConfig[j].Args.Object
		if _, isUnknown := args.(*runtime.Unknown); isUnknown {
			continue
		}
		scheme.Default(args)
	}

	// Append default configs for plugins that didn't have one explicitly set.
	for _, name := range pluginsNames(prof.Plugins) {
		if existingConfigs.Has(name) {
			continue
		}
		gvk := v1beta3.SchemeGroupVersion.WithKind(name + "Args")
		args, err := scheme.New(gvk)
		if err != nil {
			// This plugin is out-of-tree or doesn't require configuration.
			continue
		}
		scheme.Default(args)
		args.GetObjectKind().SetGroupVersionKind(gvk)
		prof.PluginConfig = append(prof.PluginConfig, v1beta3.PluginConfig{
			Name: name,
			Args: runtime.RawExtension{Object: args},
		})
	}
}

// 20个内置Plugin都配置为启用
func getDefaultPlugins() *v1beta3.Plugins {
	plugins := &v1beta3.Plugins{
		MultiPoint: v1beta3.PluginSet{
			Enabled: []v1beta3.Plugin{
				{Name: names.PrioritySort},
				{Name: names.NodeUnschedulable},
				{Name: names.NodeName},
				{Name: names.TaintToleration, Weight: pointer.Int32(3)},
				{Name: names.NodeAffinity, Weight: pointer.Int32(2)},
				{Name: names.NodePorts},
				{Name: names.NodeResourcesFit, Weight: pointer.Int32(1)},
				{Name: names.VolumeRestrictions},
				{Name: names.EBSLimits},
				{Name: names.GCEPDLimits},
				{Name: names.NodeVolumeLimits},
				{Name: names.AzureDiskLimits},
				{Name: names.VolumeBinding},
				{Name: names.VolumeZone},
				{Name: names.PodTopologySpread, Weight: pointer.Int32(2)},
				{Name: names.InterPodAffinity, Weight: pointer.Int32(2)},
				{Name: names.DefaultPreemption},
				{Name: names.NodeResourcesBalancedAllocation, Weight: pointer.Int32(1)},
				{Name: names.ImageLocality, Weight: pointer.Int32(1)},
				{Name: names.DefaultBinder},
			},
		},
	}
	// 根据Gateway启用某些Plugin
	applyFeatureGates(plugins)

	return plugins
}
```
