# syncLoop
循环的预准备

## 函数
函数接收两个参数：1. updates是PodConfig的事件通知channel，2. handler SyncHandler接口的实现类，一般为Kubelet本身
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.InfoS("Starting kubelet main sync loop")
	// The syncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
    // 获取pleg的事件Ch
	plegCh := kl.pleg.Watch()

    // 定义了三个常数用于指数补偿等待时间的计算
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	// Responsible for checking limits in resolv.conf
	// The limits do not have anything to do with individual pods
	// Since this is called in syncLoop, we don't need to call it anywhere else
    // 检查/etc/resolv.conf
	if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
		kl.dnsConfigurer.CheckLimitsForResolvConf()
	}

	for {
        // 检查Kubelet中healthCheck是否正常，如果不正常，则打印异常日志并进行指数补偿等待
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.ErrorS(err, "Skipping pod synchronization")
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
        // 在某次成功后将等待时间充值
		duration = base

        // 在执行循环逻辑之前保存当前时间，用于判断syncLoop是否正常
		kl.syncLoopMonitor.Store(kl.clock.Now())
        // 执行真正的循环逻辑
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```
