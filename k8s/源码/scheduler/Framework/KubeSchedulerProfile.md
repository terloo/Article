# KubeSchedulerProfile
KubeSchedulerProfile保存着Framework的配置信息，通过KubeSchedulerProfile可以生成Framework。默认情况下只有一个名为default-shcduler的profile

## 结构体
```go
// pkg/scheduler/apis/config/types.go
// KubeSchedulerProfile is a scheduling profile.
type KubeSchedulerProfile struct {
	// SchedulerName is the name of the scheduler associated to this profile.
	// If SchedulerName matches with the pod's "spec.schedulerName", then the pod
	// is scheduled with this profile.
	SchedulerName string

	// Plugins specify the set of plugins that should be enabled or disabled.
	// Enabled plugins are the ones that should be enabled in addition to the
	// default plugins. Disabled plugins are any of the default plugins that
	// should be disabled.
	// When no enabled or disabled plugin is specified for an extension point,
	// default plugins for that extension point will be used if there is any.
	// If a QueueSort plugin is specified, the same QueueSort Plugin and
	// PluginConfig must be specified for all profiles.
	Plugins *Plugins

	// PluginConfig is an optional set of custom plugin arguments for each plugin.
	// Omitting config args for a plugin is equivalent to using the default config
	// for that plugin.
	PluginConfig []PluginConfig
}
```
1. SchedulerName：生成的调度框架名
2. Plugins：按扩展点分类的所有插件
3. PluginConfig：插件配置
