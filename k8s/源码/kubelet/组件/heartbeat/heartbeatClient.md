# heartbeatClient
heartbeat机制使用的与api-server交互的客户端与kubelet本身所使用的客户端有所不同

## clientConfig
```go
// pkg/kubelet/kubelet.go

// 从clientConfig复制一份
heartbeatClientConfig := *clientConfig
// heartbeat的超时时间是租约到期周期和NodeStatusUpdateFrequency周期中最小值
heartbeatClientConfig.Timeout = s.KubeletConfiguration.NodeStatusUpdateFrequency.Duration
// The timeout is the minimum of the lease duration and status update frequency
leaseTimeout := time.Duration(s.KubeletConfiguration.NodeLeaseDurationSeconds) * time.Second
if heartbeatClientConfig.Timeout > leaseTimeout {
    heartbeatClientConfig.Timeout = leaseTimeout
}
// QPS为-1，禁用限流
heartbeatClientConfig.QPS = float32(-1)
// 创建
kubeDeps.HeartbeatClient, err = clientset.NewForConfig(&heartbeatClientConfig)
```