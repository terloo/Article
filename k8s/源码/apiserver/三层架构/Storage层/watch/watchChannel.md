# watchChan
watchChan从etcd中watch对象，并将接收到的对象进行处理，然后放入channel中

## 结构体
watchChan实现了`k8s.io/apimachinery/pkg/watch/watch.go#Interface`，是etcd的k8s Watch实现
```go
// k8s.io/apiserver/pkg/storage/etcd/watcher.go
type watchChan struct {
	watcher           *watcher
	key               string
	initialRev        int64
	recursive         bool
	progressNotify    bool
	internalPred      storage.SelectionPredicate
	ctx               context.Context
	cancel            context.CancelFunc
	incomingEventChan chan *event
	resultChan        chan watch.Event
	errChan           chan error
}
```
1. watcher：watcher的实例，主要使用其中保存的etcd客户端，codec等属性
2. key：watch的key
3. initialRev：初始Rev
4. recursive：是否以前缀watch
5. progressNotify：是否需要心跳事件
6. internalPred：资源对象选择器
7. incomingEventChan：接收etcd事件的channel
8. resultChan：接收处理后资源对象的channel

## run
运行此watchChan，从etcd开始接收事件，并开始消费事件
```go
// k8s.io/apiserver/pkg/storage/etcd/watcher.go
func (wc *watchChan) run() {
	watchClosedCh := make(chan struct{})
	// 生产事件
	go wc.startWatching(watchClosedCh)

	var resultChanWG sync.WaitGroup
	resultChanWG.Add(1)
	// 消费事件
	go wc.processEvent(&resultChanWG)

	select {
	case err := <-wc.errChan:
		if err == context.Canceled {
			break
		}
		errResult := transformErrorToEvent(err)
		if errResult != nil {
			// error result is guaranteed to be received by user before closing ResultChan.
			select {
			case wc.resultChan <- *errResult:
			case <-wc.ctx.Done(): // user has given up all results
			}
		}
	case <-watchClosedCh:
	case <-wc.ctx.Done(): // user cancel
	}

	// We use wc.ctx to reap all goroutines. Under whatever condition, we should stop them all.
	// It's fine to double cancel.
	wc.cancel()

	// we need to wait until resultChan wouldn't be used anymore
	resultChanWG.Wait()
	close(wc.resultChan)
}
```

## startWatching
- 如果initialRev为0，将会发送当前所有Object(创建事件)，然后从最大revision+1开始watch
- 如果initialRev不为0，将initialRev+1开始watch
```go
// 如果initialRev为0，则将initialRev设置为当前revision
// 调用etcd接口开始watch
func (wc *watchChan) startWatching(watchClosedCh chan struct{}) {
	if wc.initialRev == 0 {
		// 如果initialRev为0，调用sync操作
		if err := wc.sync(); err != nil {
			klog.Errorf("failed to sync with latest state: %v", err)
			wc.sendError(err)
			return
		}
	}

	// 直接将initalRev + 1
	opts := []clientv3.OpOption{clientv3.WithRev(wc.initialRev + 1), clientv3.WithPrevKV()}
	if wc.recursive {
		opts = append(opts, clientv3.WithPrefix())
	}
	if wc.progressNotify {
		opts = append(opts, clientv3.WithProgressNotify())
	}
	wch := wc.watcher.client.Watch(wc.ctx, wc.key, opts...)
	// 消费etcd的watchChannel
	for wres := range wch {
		if wres.Err() != nil {
			err := wres.Err()
			// If there is an error on server (e.g. compaction), the channel will return it before closed.
			logWatchChannelErr(err)
			wc.sendError(err)
			return
		}
		if wres.IsProgressNotify() {
			wc.sendEvent(progressNotifyEvent(wres.Header.GetRevision()))
			metrics.RecordEtcdBookmark(wc.watcher.objectType)
			continue
		}

		for _, e := range wres.Events {
			parsedEvent, err := parseEvent(e)
			if err != nil {
				logWatchChannelErr(err)
				wc.sendError(err)
				return
			}
			// 发送事件到incomingEventChan
			wc.sendEvent(parsedEvent)
		}
	}
	// When we come to this point, it's only possible that client side ends the watch.
	// e.g. cancel the context, close the client.
	// If this watch chan is broken and context isn't cancelled, other goroutines will still hang.
	// We should notify the main thread that this goroutine has exited.
	close(watchClosedCh)
}

// 获取最新的所有kv，并发送isCreate为true事件，给initialRev赋值为最新一次revision
func (wc *watchChan) sync() error {
	opts := []clientv3.OpOption{}
	if wc.recursive {
		opts = append(opts, clientv3.WithPrefix())
	}
	getResp, err := wc.watcher.client.Get(wc.ctx, wc.key, opts...)
	if err != nil {
		return err
	}
	wc.initialRev = getResp.Header.Revision
	for _, kv := range getResp.Kvs {
		wc.sendEvent(parseKV(kv))
	}
	return nil
}
```