# NewMap
newMap函数将不同的Profile应用在Registry上，以生成不同的Framework

## 函数
```go
// pkg/scheduler/profile/profile.go
// 使用不同的Profile来和同一个Registry来构建Framework
// NewMap builds the frameworks given by the configuration, indexed by name.
func NewMap(cfgs []config.KubeSchedulerProfile, r frameworkruntime.Registry, recorderFact RecorderFactory,
	opts ...frameworkruntime.Option) (Map, error) {
	m := make(Map)
	v := cfgValidator{m: m}

	for _, cfg := range cfgs {
        // 对每个profile都生成一个framework，并进行索引
		p, err := newProfile(cfg, r, recorderFact, opts...)
		if err != nil {
			return nil, fmt.Errorf("creating profile for scheduler name %s: %v", cfg.SchedulerName, err)
		}
		if err := v.validate(cfg, p); err != nil {
			return nil, err
		}
		m[cfg.SchedulerName] = p
	}
	return m, nil
}

func newProfile(cfg config.KubeSchedulerProfile, r frameworkruntime.Registry, recorderFact RecorderFactory,
	opts ...frameworkruntime.Option) (framework.Framework, error) {
	recorder := recorderFact(cfg.SchedulerName)
	opts = append(opts, frameworkruntime.WithEventRecorder(recorder))
    // 调用framework的初始化函数
	fwk, err := frameworkruntime.NewFramework(r, &cfg, opts...)
	if err != nil {
		return nil, err
	}
	return fwk, nil
}
```