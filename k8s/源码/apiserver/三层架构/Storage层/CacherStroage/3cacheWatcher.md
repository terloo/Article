# cacheWatcher
每一个执行了Watch操作的客户端，都会被分配一个cacheWatcher，用于接收Watch事件

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/storage/cacher/cacher.go
// cacheWatcher是非线程安全的
type cacheWatcher struct {
    // 接收事件的队列
	input     chan *watchCacheEvent
    // 推送事件的队列
	result    chan watch.Event
	done      chan struct{}
	filter    filterWithAttrsFunc
	stopped   bool
    // 终止消费的函数
	forget    func()
	versioner storage.Versioner
	// The watcher will be closed by server after the deadline,
	// save it here to send bookmark events before that.
	deadline            time.Time
	allowWatchBookmarks bool
	// Object type of the cache watcher interests
	objectType reflect.Type

	// human readable identifier that helps assigning cacheWatcher
	// instance with request
	identifier string
}
```

## 构造一个cacheWatcher
```go
// vendor/k8s.io/apiserver/storage/cacher/cacher.go
// 构造函数
func newCacheWatcher(chanSize int, filter filterWithAttrsFunc, forget func(), versioner storage.Versioner, deadline time.Time, allowWatchBookmarks bool, objectType reflect.Type, identifier string) *cacheWatcher {
	return &cacheWatcher{
		input:               make(chan *watchCacheEvent, chanSize),
		result:              make(chan watch.Event, chanSize),
		done:                make(chan struct{}),
		filter:              filter,
		stopped:             false,
		forget:              forget,
		versioner:           versioner,
		deadline:            deadline,
		allowWatchBookmarks: allowWatchBookmarks,
		objectType:          objectType,
		identifier:          identifier,
	}
}

// 客户端调用Cacher的Watch时就会返回一个cacheWatcher
func (c *Cacher) Watch(ctx context.Context, key string, opts storage.ListOptions) (watch.Interface, error) {
	pred := opts.Predicate
	watchRV, err := c.versioner.ParseResourceVersion(opts.ResourceVersion)
	if err != nil {
		return nil, err
	}

	c.ready.wait()

	triggerValue, triggerSupported := "", false
	if c.indexedTrigger != nil {
		for _, field := range pred.IndexFields {
			if field == c.indexedTrigger.indexName {
				if value, ok := pred.Field.RequiresExactMatch(field); ok {
					triggerValue, triggerSupported = value, true
				}
			}
		}
	}

	// If there is indexedTrigger defined, but triggerSupported is false,
	// we can't narrow the amount of events significantly at this point.
	//
	// That said, currently indexedTrigger is defined only for couple resources:
	// Pods, Nodes, Secrets and ConfigMaps and there is only a constant
	// number of watchers for which triggerSupported is false (excluding those
	// issued explicitly by users).
	// Thus, to reduce the risk of those watchers blocking all watchers of a
	// given resource in the system, we increase the sizes of buffers for them.
	chanSize := 10
	if c.indexedTrigger != nil && !triggerSupported {
		// TODO: We should tune this value and ideally make it dependent on the
		// number of objects of a given type and/or their churn.
		chanSize = 1000
	}

	// Determine watch timeout('0' means deadline is not set, ignore checking)
	deadline, _ := ctx.Deadline()

	identifier := fmt.Sprintf("key: %q, labels: %q, fields: %q", key, pred.Label, pred.Field)

	// Create a watcher here to reduce memory allocations under lock,
	// given that memory allocation may trigger GC and block the thread.
	// Also note that emptyFunc is a placeholder, until we will be able
	// to compute watcher.forget function (which has to happen under lock).
    // 构建一个cacheWatcher
	watcher := newCacheWatcher(chanSize, filterWithAttrsFunction(key, pred), emptyFunc, c.versioner, deadline, pred.AllowWatchBookmarks, c.objectType, identifier)

	// We explicitly use thread unsafe version and do locking ourself to ensure that
	// no new events will be processed in the meantime. The watchCache will be unlocked
	// on return from this function.
	// Note that we cannot do it under Cacher lock, to avoid a deadlock, since the
	// underlying watchCache is calling processEvent under its lock.
	c.watchCache.RLock()
	defer c.watchCache.RUnlock()
	initEvents, err := c.watchCache.GetAllEventsSinceThreadUnsafe(watchRV)
	if err != nil {
		// To match the uncached watch implementation, once we have passed authn/authz/admission,
		// and successfully parsed a resource version, other errors must fail with a watch event of type ERROR,
		// rather than a directly returned error.
		return newErrWatcher(err), nil
	}

	func() {
		c.Lock()
		defer c.Unlock()
		// Update watcher.forget function once we can compute it.
		watcher.forget = forgetWatcher(c, c.watcherIdx, triggerValue, triggerSupported)
        // 将cacheWatcher存储到Cacher中进行统一管理
        // id为c.watcherIdx，每添加一个cacheWatcher就会使id++
		c.watchers.addWatcher(watcher, c.watcherIdx, triggerValue, triggerSupported)

		// Add it to the queue only when the client support watch bookmarks.
		if watcher.allowWatchBookmarks {
			c.bookmarkWatchers.addWatcher(watcher)
		}
		c.watcherIdx++
	}()

    // 开始监听事件，并传入一个初始化事件
	go watcher.processEvents(ctx, initEvents, watchRV)
	return watcher, nil
}
```

## 处理事件的循环
```go
// vendor/k8s.io/apiserver/storage/cacher/cacher.go
// 这个方式被Cacher使用goroutine启动
func (c *cacheWatcher) process(ctx context.Context, resourceVersion uint64) {
	// At this point we already start processing incoming watch events.
	// However, the init event can still be processed because their serialization
	// and sending to the client happens asynchrnously.
	// TODO: As describe in the KEP, we would like to estimate that by delaying
	//   the initialization signal proportionally to the number of events to
	//   process, but we're leaving this to the tuning phase.
	utilflowcontrol.WatchInitialized(ctx)

	defer close(c.result)
	defer c.Stop()
	for {
		select {
		// 从input队列获取event
		case event, ok := <-c.input:
			if !ok {
				return
			}
			// only send events newer than resourceVersion
            // 仅处理(事件资源版本 > 开始循环时指定的资源版本)的事件
			if event.ResourceVersion > resourceVersion {
				c.sendWatchCacheEvent(event)
			}
		case <-ctx.Done():
			return
		}
	}
}
``` 