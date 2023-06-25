# Plugins
Plugins保存了所有Plugin的工厂函数，在合适的时机实例化Plugin  
通过NewFromPlugins方法来将所有应开启的Plugin整合为一个Plugin

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/admission/plugins.go
type Plugins struct {
    // 保护registry的并发性
	lock     sync.Mutex
    // 懒加载工厂
    // 以map的形式保存所有AdmissionController工厂函数
	registry map[string]Factory
}

// Factory定了获取Plugin的工厂函数
// config参数是AdmissionCotroller的配置文件信息(通过--admission-control-config-file选项传入)
type Factory func(config io.Reader) (Interface, error)
```

## 构造函数
```go
// vendor/k8s.io/apiserver/pkg/admission/plugins.go
func NewPlugins() *Plugins {
	return &Plugins{}
}
```

## Register
将一个Plugin的工厂函数注册到Plugins中
```go
// vendor/k8s.io/apiserver/pkg/admission/plugins.go
// 如果已注册，则报错退出
func (ps *Plugins) Register(name string, plugin Factory) {
	ps.lock.Lock()
	defer ps.lock.Unlock()
	if ps.registry != nil {
		_, found := ps.registry[name]
		if found {
			klog.Fatalf("Admission plugin %q was registered twice", name)
		}
	} else {
		ps.registry = map[string]Factory{}
	}

	klog.V(1).InfoS("Registered admission plugin", "plugin", name)
	ps.registry[name] = plugin
}
```

## getPlugin
通过工厂函数实例化Plugin
```go
// vendor/k8s.io/apiserver/pkg/admission/plugins.go
// bool参数代表是否有对应的工厂函数
func (ps *Plugins) getPlugin(name string, config io.Reader) (Interface, bool, error) {
	ps.lock.Lock()
	defer ps.lock.Unlock()
	f, found := ps.registry[name]
	if !found {
		return nil, false, nil
	}

	config1, config2, err := splitStream(config)
	if err != nil {
		return nil, true, err
	}
    // 在测试时使用，默认返回true
	if !PluginEnabledFn(name, config1) {
		return nil, true, nil
	}

    // 调用工厂函数实例化该Plugin
	ret, err := f(config2)
	return ret, true, err
}
```

## InitPlugin
实例化工厂函数并使用Initalizer进行初始化
```go
// vendor/k8s.io/apiserver/pkg/admission/plugins.go
func (ps *Plugins) InitPlugin(name string, config io.Reader, pluginInitializer PluginInitializer) (Interface, error) {
	if name == "" {
		klog.Info("No admission plugin specified.")
		return nil, nil
	}

    // 实例化Plugin
	plugin, found, err := ps.getPlugin(name, config)
	if err != nil {
		return nil, fmt.Errorf("couldn't init admission plugin %q: %v", name, err)
	}
	if !found {
		return nil, fmt.Errorf("unknown admission plugin: %s", name)
	}

    // 初始化Plugin
	pluginInitializer.Initialize(plugin)
	// ensure that plugins have been properly initialized
    // 确保已经初始化完毕，字段已正确填充
	if err := ValidateInitialization(plugin); err != nil {
		return nil, fmt.Errorf("failed to initialize admission plugin %q: %v", name, err)
	}

	return plugin, nil
}
```