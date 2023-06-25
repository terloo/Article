# objectCacheItem
objectCacheItem资源对象对应的reflector和资源对象的本地缓存cacheStore

## 结构体
```go
// pkg/kubelet/util/manager/watch_based_manager.go
type objectCacheItem struct {
	refCount  int
	store     *cacheStore
	reflector *cache.Reflector

	hasSynced func() (bool, error)

	// waitGroup is used to ensure that there won't be two concurrent calls to reflector.Run
	waitGroup sync.WaitGroup

	// lock is to ensure the access and modify of lastAccessTime, stopped, and immutable are thread safety,
	// and protecting from closing stopCh multiple times.
	lock           sync.Mutex
	lastAccessTime time.Time
	stopped        bool
	immutable      bool
	stopCh         chan struct{}
}
```
1. refCount：记录该对象的引用数
2. store：资源对象本地缓存结构，cacheStore是Indexer的扩展类
3. reflector：Reflector，Reflector获取的数据会被缓存到store中
4. waitGroup：用于确保reflector.Run不会被并发调用
5. lock：保护lastAccessTime

## 方法
```go
// pkg/kubelet/util/manager/watch_based_manager.go

// 启动该item的reflector
func (i *objectCacheItem) startReflector() {
	i.waitGroup.Wait()
	i.waitGroup.Add(1)
	defer i.waitGroup.Done()
	i.reflector.Run(i.stopCh)
}

// 线程安全的停止该item的reflector
func (i *objectCacheItem) stop() bool {
	i.lock.Lock()
	defer i.lock.Unlock()
	return i.stopThreadUnsafe()
}

// 非线程安全的停止该item的reflector
func (i *objectCacheItem) stopThreadUnsafe() bool {
	// 如果已经停止了，那么返回false
	if i.stopped {
		return false
	}
	i.stopped = true
	close(i.stopCh)
	// 如果该item是可变的，那么将其重新标记为未初始化
	if !i.immutable {
		i.store.unsetInitialized()
	}
	return true
}

func (i *objectCacheItem) setLastAccessTime(time time.Time) {
	i.lock.Lock()
	defer i.lock.Unlock()
	i.lastAccessTime = time
}

func (i *objectCacheItem) setImmutable() {
	i.lock.Lock()
	defer i.lock.Unlock()
	i.immutable = true
}

func (i *objectCacheItem) stopIfIdle(now time.Time, maxIdleTime time.Duration) bool {
	i.lock.Lock()
	defer i.lock.Unlock()
	// Ensure that we don't try to stop not yet initialized reflector.
	// In case of overloaded kube-apiserver, if the list request is
	// already being processed, all the work would lost and would have
	// to be retried.
	// 如果距离上次lastAccessTime已经超过了maxIdleTime时间，将item停止
	if !i.stopped && i.store.hasSynced() && now.After(i.lastAccessTime.Add(maxIdleTime)) {
		return i.stopThreadUnsafe()
	}
	return false
}

// 重启Reflector
func (i *objectCacheItem) restartReflectorIfNeeded() {
	i.lock.Lock()
	defer i.lock.Unlock()
	if i.immutable || !i.stopped {
		return
	}
	i.stopCh = make(chan struct{})
	i.stopped = false
	go i.startReflector()
}

```