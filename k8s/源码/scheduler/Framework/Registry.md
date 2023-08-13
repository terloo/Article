# Registry
所有等待被应用到Framework中的Plugin会被统一注册到Registry中，多个Registry可以进行合并操作  
Framework使用Registry来进行插件的生成和初始化操作，在Framework初始化之前，所有的Plugin都必须被放置于一个Registry之中

## 结构体
```go
// pkg/scheduler/framework/runtime/registry.go
// 从插件名映射到生成插件的函数
type Registry map[string]PluginFactory

// 生成插件的函数
type PluginFactory = func(configuration runtime.Object, f framework.Handle) (framework.Plugin, error)
```

## Options
Scheudler定义了一个用于修改Registry的Option函数，专门用于让用户添加外部调度插件
```go
// cmd/kube-scheduler/app/server.go
type Option func(runtime.Registry) error
```

## 方法
```go
// pkg/scheduler/framework/runtime/registry.go
// Register adds a new plugin to the registry. If a plugin with the same name
// exists, it returns an error.
// 注册一个PluginFramework
func (r Registry) Register(name string, factory PluginFactory) error {
	if _, ok := r[name]; ok {
		return fmt.Errorf("a plugin named %v already exists", name)
	}
	r[name] = factory
	return nil
}

// 取消注册一个PluginFramework
// Unregister removes an existing plugin from the registry. If no plugin with
// the provided name exists, it returns an error.
func (r Registry) Unregister(name string) error {
	if _, ok := r[name]; !ok {
		return fmt.Errorf("no plugin named %v exists", name)
	}
	delete(r, name)
	return nil
}

// 将传入的Registry合并到该Regsitry中
// Merge merges the provided registry to the current one.
func (r Registry) Merge(in Registry) error {
	for name, factory := range in {
		if err := r.Register(name, factory); err != nil {
			return err
		}
	}
	return nil
}
```

## 内置Regsitry
可以通过WithFrameworkOutOfTreeRegistry(需要重新编译Scheduler)可以注册额外的Plugins
```go
// pkg/scheduler/framework/plugins/registry.go
// NewInTreeRegistry builds the registry with all the in-tree plugins.
// A scheduler that runs out of tree plugins can register additional plugins
// through the WithFrameworkOutOfTreeRegistry option.
func NewInTreeRegistry() runtime.Registry {
    // 根据gateway来决定某些插件是否启用
	fts := plfeature.Features{
		EnablePodAffinityNamespaceSelector: feature.DefaultFeatureGate.Enabled(features.PodAffinityNamespaceSelector),
		EnablePodDisruptionBudget:          feature.DefaultFeatureGate.Enabled(features.PodDisruptionBudget),
		EnablePodOverhead:                  feature.DefaultFeatureGate.Enabled(features.PodOverhead),
		EnableReadWriteOncePod:             feature.DefaultFeatureGate.Enabled(features.ReadWriteOncePod),
		EnableVolumeCapacityPriority:       feature.DefaultFeatureGate.Enabled(features.VolumeCapacityPriority),
		EnableCSIStorageCapacity:           feature.DefaultFeatureGate.Enabled(features.CSIStorageCapacity),
	}

	return runtime.Registry{
		selectorspread.Name:                  selectorspread.New,
		imagelocality.Name:                   imagelocality.New,
		tainttoleration.Name:                 tainttoleration.New,
		nodename.Name:                        nodename.New,
		nodeports.Name:                       nodeports.New,
		nodeaffinity.Name:                    nodeaffinity.New,
		podtopologyspread.Name:               podtopologyspread.New,
		nodeunschedulable.Name:               nodeunschedulable.New,
		noderesources.FitName:                runtime.FactoryAdapter(fts, noderesources.NewFit),
		noderesources.BalancedAllocationName: runtime.FactoryAdapter(fts, noderesources.NewBalancedAllocation),
		volumebinding.Name:                   runtime.FactoryAdapter(fts, volumebinding.New),
		volumerestrictions.Name:              runtime.FactoryAdapter(fts, volumerestrictions.New),
		volumezone.Name:                      volumezone.New,
		nodevolumelimits.CSIName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewCSI),
		nodevolumelimits.EBSName:             runtime.FactoryAdapter(fts, nodevolumelimits.NewEBS),
		nodevolumelimits.GCEPDName:           runtime.FactoryAdapter(fts, nodevolumelimits.NewGCEPD),
		nodevolumelimits.AzureDiskName:       runtime.FactoryAdapter(fts, nodevolumelimits.NewAzureDisk),
		nodevolumelimits.CinderName:          runtime.FactoryAdapter(fts, nodevolumelimits.NewCinder),
		interpodaffinity.Name:                runtime.FactoryAdapter(fts, interpodaffinity.New),
		queuesort.Name:                       queuesort.New,
		defaultbinder.Name:                   defaultbinder.New,
		defaultpreemption.Name:               runtime.FactoryAdapter(fts, defaultpreemption.New),
	}
}
```