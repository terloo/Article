# Cacher
Cacher是对UnderlyingSrotage的封装，与UnderlyingStorage一样，实现了storage.Interface接口，是对UnderlyingStorage的装饰。

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/stotage/cacher/cacher.go
type Cacher struct {
	// HighWaterMarks for performance debugging.
	// Important: Since HighWaterMark is using sync/atomic, it has to be at the top of the struct due to a bug on 32-bit platforms
	// See: https://golang.org/pkg/sync/atomic/ for more information
	incomingHWM storage.HighWaterMark
	// Incoming events that should be dispatched to watchers.
	incoming chan watchCacheEvent

	resourcePrefix string

	sync.RWMutex

	// Before accessing the cacher's cache, wait for the ready to be ok.
	// This is necessary to prevent users from accessing structures that are
	// uninitialized or are being repopulated right now.
	// ready needs to be set to false when the cacher is paused or stopped.
	// ready needs to be set to true when the cacher is ready to use after
	// initialization.
	ready *ready

	// Underlying storage.Interface.
    // UnderlyingStorage
	storage storage.Interface

	// Expected type of objects in the underlying cache.
	objectType reflect.Type

	// "sliding window" of recent changes of objects and the current state.
    // 滑动窗口的实现
	watchCache *watchCache
	reflector  *cache.Reflector

	// Versioner is used to handle resource versions.
	versioner storage.Versioner

	// newFunc is a function that creates new empty object storing a object of type Type.
	newFunc func() runtime.Object

	// indexedTrigger is used for optimizing amount of watchers that needs to process
	// an incoming event.
	indexedTrigger *indexedTriggerFunc
	// watchers is mapping from the value of trigger function that a
	// watcher is interested into the watchers
	watcherIdx int
	watchers   indexedWatchers

	// Defines a time budget that can be spend on waiting for not-ready watchers
	// while dispatching event before shutting them down.
	dispatchTimeoutBudget timeBudget

	// Handling graceful termination.
	stopLock sync.RWMutex
	stopped  bool
	stopCh   chan struct{}
	stopWg   sync.WaitGroup

	clock clock.Clock
	// timer is used to avoid unnecessary allocations in underlying watchers.
	timer *time.Timer

	// dispatching determines whether there is currently dispatching of
	// any event in flight.
	dispatching bool
	// watchersBuffer is a list of watchers potentially interested in currently
	// dispatched event.
	watchersBuffer []*cacheWatcher
	// blockedWatchers is a list of watchers whose buffer is currently full.
	blockedWatchers []*cacheWatcher
	// watchersToStop is a list of watchers that were supposed to be stopped
	// during current dispatching, but stopping was deferred to the end of
	// dispatching that event to avoid race with closing channels in watchers.
	watchersToStop []*cacheWatcher
	// Maintain a timeout queue to send the bookmark event before the watcher times out.
	bookmarkWatchers *watcherBookmarkTimeBuckets
}
```