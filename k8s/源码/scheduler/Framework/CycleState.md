# CycleState
CycleState结构体可以让调度插件保存任意其在调度流程需要的数据，用以在接下来的扩展点中进行使用。保存的数据也可以被其他插件所见和修改。线程安全

## 结构体
```go
// pkg/scheduler/framework/cycle_state.go
// CycleState provides a mechanism for plugins to store and retrieve arbitrary data.
// StateData stored by one plugin can be read, altered, or deleted by another plugin.
// CycleState does not provide any data protection, as all plugins are assumed to be
// trusted.
type CycleState struct {
	mx      sync.RWMutex
	storage map[StateKey]StateData
	// if recordPluginMetrics is true, PluginExecutionDuration will be recorded for this cycle.
	recordPluginMetrics bool
}

// StateData由插件实现并进行使用
// StateData is a generic type for arbitrary data stored in CycleState.
type StateData interface {
	// Clone is an interface to make a copy of StateData. For performance reasons,
	// clone should make shallow copies for members (e.g., slices or maps) that are not
	// impacted by PreFilter's optional AddPod/RemovePod methods.
	Clone() StateData
}

// Key一般是<阶段名><插件名>
// StateKey is the type of keys stored in CycleState.
type StateKey string
```

## 构造函数
```go
// pkg/scheduler/framework/cycle_state.go
// NewCycleState initializes a new CycleState and returns its pointer.
func NewCycleState() *CycleState {
	return &CycleState{
		storage: make(map[StateKey]StateData),
	}
}
```

## 增删拷贝
```go
// pkg/scheduler/framework/cycle_state.go
// Clone creates a copy of CycleState and returns its pointer. Clone returns
// nil if the context being cloned is nil.
func (c *CycleState) Clone() *CycleState {
	if c == nil {
		return nil
	}
	copy := NewCycleState()
	for k, v := range c.storage {
		copy.Write(k, v.Clone())
	}
	return copy
}

// Read retrieves data with the given "key" from CycleState. If the key is not
// present an error is returned.
// This function is thread safe by acquiring an internal lock first.
func (c *CycleState) Read(key StateKey) (StateData, error) {
	c.mx.RLock()
	defer c.mx.RUnlock()
	if v, ok := c.storage[key]; ok {
		return v, nil
	}
	return nil, ErrNotFound
}

// Write stores the given "val" in CycleState with the given "key".
// This function is thread safe by acquiring an internal lock first.
func (c *CycleState) Write(key StateKey, val StateData) {
	c.mx.Lock()
	c.storage[key] = val
	c.mx.Unlock()
}

// Delete deletes data with the given key from CycleState.
// This function is thread safe by acquiring an internal lock first.
func (c *CycleState) Delete(key StateKey) {
	c.mx.Lock()
	delete(c.storage, key)
	c.mx.Unlock()
}
```