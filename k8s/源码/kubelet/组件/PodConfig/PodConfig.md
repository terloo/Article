# PodConfig
PodConfig监控着所有可能的Pod配置来源，由Mux和podStorage实现。处理Pod资源的主动性改动

## 结构体
```go
// pkg/kubelet/config/config.go
type PodConfig struct {
	pods *podStorage
	mux  *config.Mux

	// the channel of denormalized changes passed to listeners
	updates chan kubetypes.PodUpdate

	// contains the list of all configured sources
	sourcesLock sync.Mutex
	sources     sets.String
}
```
1. pods：podStorage
2. mux：包装了podStorage的mux
3. sources：来源名的Set集合
4. sourcesLock：维护sources字段的锁

## 构建函数
```go
// pkg/kubelet/config/config.go
func NewPodConfig(mode PodConfigNotificationMode, recorder record.EventRecorder) *PodConfig {
	updates := make(chan kubetypes.PodUpdate, 50)
	storage := newPodStorage(updates, mode, recorder)
	podConfig := &PodConfig{
		pods:    storage,
		mux:     config.NewMux(storage),
		updates: updates,
		sources: sets.String{},
	}
	return podConfig
}

func newPodStorage(updates chan<- kubetypes.PodUpdate, mode PodConfigNotificationMode, recorder record.EventRecorder) *podStorage {
	return &podStorage{
		pods:        make(map[string]map[types.UID]*v1.Pod),
		mode:        mode,
		updates:     updates,
		sourcesSeen: sets.String{},
		recorder:    recorder,
	}
}
```

## Channel
```go
获取来源channel，source向其生产podUpdate
// pkg/kubelet/config/config.go
func (c *PodConfig) Channel(source string) chan<- interface{} {
	c.sourcesLock.Lock()
	defer c.sourcesLock.Unlock()
	c.sources.Insert(source)
	// 调用mux的Channel获取来源channel
	return c.mux.Channel(source)
}
```

## Updates
获取updates channel
```go
func (c *PodConfig) Updates() <-chan kubetypes.PodUpdate {
	return c.updates
}
```

## SeenAllSources
健康检查，查看传入的source集合是否都已经开始进行事件传递
```go
// pkg/kubelet/config/config.go
func (c *PodConfig) SeenAllSources(seenSources sets.String) bool {
	if c.pods == nil {
		return false
	}
	klog.V(5).InfoS("Looking for sources, have seen", "sources", c.sources.List(), "seenSources", seenSources)
	return seenSources.HasAll(c.sources.List()...) && c.pods.seenSources(c.sources.List()...)
}
```

## Sync
调用podStorage的Sync方法，使用SET全量推送所有来源的所有数据，暂未使用
```go
// pkg/kubelet/config/config.go
func (c *PodConfig) Sync() {
	c.pods.Sync()
}
```
