# watcher
watcher是storage层中用于创建watch管道的结构体

## 结构体
```go
// k8s.io/apiserver/pkg/storage/etcd3/watcher.go
type watcher struct {
	client      *clientv3.Client
	codec       runtime.Codec
	newFunc     func() runtime.Object
	objectType  string
	versioner   storage.Versioner
	transformer value.Transformer
}
```
1. client：etcd客户端
2. versioner：用于获取资源对象在etcd中的版本

## Watch
接收一个key，返回watch.Interface
```go
// k8s.io/apiserver/pkg/storage/etcd3/watcher.go
func (w *watcher) Watch(ctx context.Context, key string, rev int64, recursive, progressNotify bool, pred storage.SelectionPredicate) (watch.Interface, error) {
	if recursive && !strings.HasSuffix(key, "/") {
		key += "/"
	}
    // 获取watchChannal实例
	wc := w.createWatchChan(ctx, key, rev, recursive, progressNotify, pred)
    // 运行watchChannel实例
	go wc.run()

	// For etcd watch we don't have an easy way to answer whether the watch
	// has already caught up. So in the initial version (given that watchcache
	// is by default enabled for all resources but Events), we just deliver
	// the initialization signal immediately. Improving this will be explored
	// in the future.
	utilflowcontrol.WatchInitialized(ctx)

	return wc, nil
}

func (w *watcher) createWatchChan(ctx context.Context, key string, rev int64, recursive, progressNotify bool, pred storage.SelectionPredicate) *watchChan {
	wc := &watchChan{
		watcher:           w,
		key:               key,
		initialRev:        rev,
		recursive:         recursive,
		progressNotify:    progressNotify,
		internalPred:      pred,
		incomingEventChan: make(chan *event, incomingBufSize),
		resultChan:        make(chan watch.Event, outgoingBufSize),
		errChan:           make(chan error, 1),
	}
	if pred.Empty() {
		// The filter doesn't filter out any object.
		wc.internalPred = storage.Everything
	}

	// The etcd server waits until it cannot find a leader for 3 election
	// timeouts to cancel existing streams. 3 is currently a hard coded
	// constant. The election timeout defaults to 1000ms. If the cluster is
	// healthy, when the leader is stopped, the leadership transfer should be
	// smooth. (leader transfers its leadership before stopping). If leader is
	// hard killed, other servers will take an election timeout to realize
	// leader lost and start campaign.
	wc.ctx, wc.cancel = context.WithCancel(clientv3.WithRequireLeader(ctx))
	return wc
}
```
