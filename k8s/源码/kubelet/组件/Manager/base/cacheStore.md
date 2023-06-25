# cacheStore
cacheStore是cache.Store(此处实现类一般为Indexer)的扩展类  
重写了Replace方法，在Replace时做了个标记，用于分辨该store是否已经同步完成

## 结构体
```go
// pkg/kubelet/util/manager/watch_based_manager.go
type cacheStore struct {
	cache.Store
	lock        sync.Mutex
	initialized bool
}
```

## 方法
```go
// pkg/kubelet/util/manager/watch_based_manager.go

// 在Replace成功后，将其标记记为true
func (c *cacheStore) Replace(list []interface{}, resourceVersion string) error {
	c.lock.Lock()
	defer c.lock.Unlock()
	err := c.Store.Replace(list, resourceVersion)
	if err != nil {
		return err
	}
	c.initialized = true
	return nil
}

// 判断是否同步完成
func (c *cacheStore) hasSynced() bool {
	c.lock.Lock()
	defer c.lock.Unlock()
	return c.initialized
}

// 标记为未初始化，该store可以初始化后重新使用
func (c *cacheStore) unsetInitialized() {
	c.lock.Lock()
	defer c.lock.Unlock()
	c.initialized = false
}
```