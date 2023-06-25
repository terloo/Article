# SourceApiserver
ApiserverSource在apiServer每次发生Pod变动时，都会将全量数据以Set的类型推送到podConfig中

## 函数
```go
// pkg/kubelet/config/apiserver.go
const WaitForAPIServerSyncPeriod = 1 * time.Second

// NewSourceApiserver creates a config source that watches and pulls from the apiserver.
func NewSourceApiserver(c clientset.Interface, nodeName types.NodeName, nodeHasSynced func() bool, updates chan<- interface{}) {
    // 获取本node上所有的pod
	lw := cache.NewListWatchFromClient(c.CoreV1().RESTClient(), "pods", metav1.NamespaceAll, fields.OneTermEqualSelector("spec.nodeName", string(nodeName)))

	// The Reflector responsible for watching pods at the apiserver should be run only after
	// the node sync with the apiserver has completed.
	klog.InfoS("Waiting for node sync before watching apiserver pods")
	go func() {
		for {
            // 先等待node的informer同步完成
			if nodeHasSynced() {
				klog.V(4).InfoS("node sync completed")
				break
			}
			time.Sleep(WaitForAPIServerSyncPeriod)
			klog.V(4).InfoS("node sync has not completed yet")
		}
		klog.InfoS("Watching apiserver")
        // node同步完成后执行
		newSourceApiserverFromLW(lw, updates)
	}()
}

// newSourceApiserverFromLW holds creates a config source that watches and pulls from the apiserver.
func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) {
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		// 将全部数据推送到SET
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
	}
    // 用UndeltaStore来创建一个Reflector，UndeltaStore每次被进行增删改查时，将会List出全部数据执行send函数
	r := cache.NewReflector(lw, &v1.Pod{}, cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc), 0)
	go r.Run(wait.NeverStop)
}
```