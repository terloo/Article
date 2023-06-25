# Options
kubeapiserver中与Admission相关的所有选项

## 开启关闭Plugins的计算方法总结
1. --admission-control与--disable-admission-plugins/--enable-admission-plugins无法同时配置
2. --admission-control选项配置的插件为显式开启，其余为显式关闭
3. --disable-admission-plugins配置的插件为显式关闭
4. --enable-admission-plugins配置的插件为显式开启
5. 去重(显式关闭 + 默认关闭) - 显示开启 = 最终关闭的插件
6. 所有插件 - 最终关闭的插件 = 最终开启的插件

## 结构体
```go
// pkg/kubeapiserver/options/admission.go
// 一个兼容老配置Options的结构体，其主体是genericoptions.AdmissionOptions
type AdmissionOptions struct {
	// GenericAdmission holds the generic admission options.
	GenericAdmission *genericoptions.AdmissionOptions
	// DEPRECATED flag, should use EnabledAdmissionPlugins and DisabledAdmissionPlugins.
	// They are mutually exclusive, specify both will lead to an error.
	// 兼容--admission-control这个选项的字段，该选项已过期，同时指定该options和其他新options会产生error
	PluginNames []string
}

// vendor/k8s.io/apiserver/pkg/options/admission.go
// 新AdmissionOptions结构体
type AdmissionOptions struct {
	// RecommendedPluginOrder holds an ordered list of plugin names we recommend to use by default
	RecommendedPluginOrder []string
	// DefaultOffPlugins is a set of plugin names that is disabled by default
	DefaultOffPlugins sets.String

	// EnablePlugins indicates plugins to be enabled passed through `--enable-admission-plugins`.
	EnablePlugins []string
	// DisablePlugins indicates plugins to be disabled passed through `--disable-admission-plugins`.
	DisablePlugins []string
	// ConfigFile is the file path with admission control configuration.
	ConfigFile string
	// Plugins contains all registered plugins.
	Plugins *admission.Plugins
	// Decorators is a list of admission decorator to wrap around the admission plugins
	Decorators admission.Decorators
}
```
1. RecommendedPluginOrder：所有AdmissionPlugin的循序排列，用于排序
2. DefaultOffPlugins：所有默认关闭的AdmissionPlugin
3. EnablePlugins：接受--enable-admission-plugins选项传入的Plugins
4. DisablePlugins：接受--disable-admission-plugins选项传入的Plugins
5. ConfigFile：接受--admission-control-config-file选项传入的文件路径
6. Plugins：存储所有已注册的Plugin，可以将所有已开启的Plugins整合为一个Plugin
7. Decorators：[]Decorator，所有AdmissionPlugin的包装函数

## 构造函数
```go
// pkg/kubeapiserver/options/admission.go
func NewAdmissionOptions() *AdmissionOptions {
	// 先构建通用的AdmissionOptions
	options := genericoptions.NewAdmissionOptions()
	// 注册其余的Plugin
	RegisterAllAdmissionPlugins(options.Plugins)
	// 重新修改Plugins顺序
	options.RecommendedPluginOrder = AllOrderedPlugins
	// 设置所有默认关闭的Admission
	options.DefaultOffPlugins = DefaultOffAdmissionPlugins()

	return &AdmissionOptions{
		GenericAdmission: options,
	}
}

// vendor/k8s.io/apiserver/pkg/server/options/admission.go
func NewAdmissionOptions() *AdmissionOptions {
	options := &AdmissionOptions{
		Plugins:    admission.NewPlugins(),
		Decorators: admission.Decorators{admission.DecoratorFunc(admissionmetrics.WithControllerMetrics)},
		// This list is mix of mutating admission plugins and validating
		// admission plugins. The apiserver always runs the validating ones
		// after all the mutating ones, so their relative order in this list
		// doesn't matter.
		RecommendedPluginOrder: []string{lifecycle.PluginName, mutatingwebhook.PluginName, validatingwebhook.PluginName},
		DefaultOffPlugins:      sets.NewString(),
	}
	// 通用的AdmissionOptions注册了lifecycle，mutatingwebhook，validatingwebhook三个Plugin
	server.RegisterAllAdmissionPlugins(options.Plugins)
	return options
}
```

## AddFlags
```go
// pkg/apiserver/options/admission.go
func (a *AdmissionOptions) AddFlags(fs *pflag.FlagSet) {
	fs.StringSliceVar(&a.PluginNames, "admission-control", a.PluginNames, ""+
		"Admission is divided into two phases. "+
		"In the first phase, only mutating admission plugins run. "+
		"In the second phase, only validating admission plugins run. "+
		"The names in the below list may represent a validating plugin, a mutating plugin, or both. "+
		"The order of plugins in which they are passed to this flag does not matter. "+
		"Comma-delimited list of: "+strings.Join(a.GenericAdmission.Plugins.Registered(), ", ")+".")
	fs.MarkDeprecated("admission-control", "Use --enable-admission-plugins or --disable-admission-plugins instead. Will be removed in a future version.")
	fs.Lookup("admission-control").Hidden = false

	a.GenericAdmission.AddFlags(fs)
}

// vendor/k8s.io/apiserver/pkg/server/options/admission.go
func (a *AdmissionOptions) AddFlags(fs *pflag.FlagSet) {
	if a == nil {
		return
	}

	fs.StringSliceVar(&a.EnablePlugins, "enable-admission-plugins", a.EnablePlugins, ""+
		"admission plugins that should be enabled in addition to default enabled ones ("+
		strings.Join(a.defaultEnabledPluginNames(), ", ")+"). "+
		"Comma-delimited list of admission plugins: "+strings.Join(a.Plugins.Registered(), ", ")+". "+
		"The order of plugins in this flag does not matter.")
	fs.StringSliceVar(&a.DisablePlugins, "disable-admission-plugins", a.DisablePlugins, ""+
		"admission plugins that should be disabled although they are in the default enabled plugins list ("+
		strings.Join(a.defaultEnabledPluginNames(), ", ")+"). "+
		"Comma-delimited list of admission plugins: "+strings.Join(a.Plugins.Registered(), ", ")+". "+
		"The order of plugins in this flag does not matter.")
	fs.StringVar(&a.ConfigFile, "admission-control-config-file", a.ConfigFile,
		"File with admission control configuration.")
}
```

## ApplyTo
生成最终的Plugin，并将其应用到genericserver.Config的AdmissionControl字段中
```go
// pkg/kubeapiserver/options/admission.go
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeAPIServerClientConfig *rest.Config,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	if a == nil {
		return nil
	}

	// 如果使用了老选项配置，将其解析为新选项配置
	if a.PluginNames != nil {
		// pass PluginNames to generic AdmissionOptions
		// PluginNames中的Plugins开启，其余Plugins关闭
		a.GenericAdmission.EnablePlugins, a.GenericAdmission.DisablePlugins = computePluginNames(a.PluginNames, a.GenericAdmission.RecommendedPluginOrder)
	}

	// 调用新选项配置的ApplyTo
	return a.GenericAdmission.ApplyTo(c, informers, kubeAPIServerClientConfig, features, pluginInitializers...)
}

func computePluginNames(explicitlyEnabled []string, all []string) (enabled []string, disabled []string) {
	return explicitlyEnabled, sets.NewString(all...).Difference(sets.NewString(explicitlyEnabled...)).List()
}

// vendor/k8s.io/apiserver/pkg/server/options/admission.go
// 传入的PluginInitializers包含webhook和k8s内部Plugin的初始化器
func (a *AdmissionOptions) ApplyTo(
	c *server.Config,
	informers informers.SharedInformerFactory,
	kubeAPIServerClientConfig *rest.Config,
	features featuregate.FeatureGate,
	pluginInitializers ...admission.PluginInitializer,
) error {
	if a == nil {
		return nil
	}

	// Admission depends on CoreAPI to set SharedInformerFactory and ClientConfig.
	if informers == nil {
		return fmt.Errorf("admission depends on a Kubernetes core API shared informer, it cannot be nil")
	}

	// 用于计算最终需要开启的Plugins的函数
	pluginNames := a.enabledPluginNames()

	pluginsConfigProvider, err := admission.ReadAdmissionConfiguration(pluginNames, a.ConfigFile, configScheme)
	if err != nil {
		return fmt.Errorf("failed to read plugin config: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(kubeAPIServerClientConfig)
	if err != nil {
		return err
	}
	genericInitializer := initializer.New(clientset, informers, c.Authorization.Authorizer, features)
	initializersChain := admission.PluginInitializers{}
	pluginInitializers = append(pluginInitializers, genericInitializer)
	// 向pluginInitializers追加一个通用初始化器，然后重新封装为PluginInitializers
	initializersChain = append(initializersChain, pluginInitializers...)

	// 将PluginInitializers传入Plugins结构体中用于初始化Plugin
	admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
	if err != nil {
		return err
	}

	// 将最终获取的admissionChain存储到genericServer的Config中
	c.AdmissionControl = admissionmetrics.WithStepMetrics(admissionChain)
	return nil
}

// 计算哪些Plugins应该开启的最终函数
func (a *AdmissionOptions) enabledPluginNames() []string {
	allOffPlugins := append(a.DefaultOffPlugins.List(), a.DisablePlugins...)
	// 所有应当关闭的Plugins
	disabledPlugins := sets.NewString(allOffPlugins...)
	// 所有应当开启的Plugins
	enabledPlugins := sets.NewString(a.EnablePlugins...)
	// 如果一个Plugin同时配置了开启关闭，则从应该关闭的Plugins中将其剔除
	disabledPlugins = disabledPlugins.Difference(enabledPlugins)

	// 按默认顺序重新整理顺序，如果Plugin不在应该关闭的Plugins中，则认为其开启
	orderedPlugins := []string{}
	for _, plugin := range a.RecommendedPluginOrder {
		if !disabledPlugins.Has(plugin) {
			orderedPlugins = append(orderedPlugins, plugin)
		}
	}

	return orderedPlugins
}
```
