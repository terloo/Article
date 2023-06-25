# heartbeat
kubelet的heartbeat分为两部分
1. 通过heartbeatClient来更新cordination.k8s.io/v1/namespaces/kube-node-lease/leases/<nodeName>资源对象，nodeLifeCycleManger将会已此为依据来更新node的就绪状态
2. 通过kubelet的syncNodeStatus()方法来将节点状态更新到Node资源对象

## 配置
| 配置名                    | 默认值 | 说明                                                      |
| ------------------------- | ------ | --------------------------------------------------------- |
| NodeStatusUpdateFrequency | 10s    | kubelet计算节点状态的频率                                 |
| NodeStatusReportFrequency | 5m     | kubelet在节点状态未改变时，同步到api-server的最长等待时间 |


## 启动heartbeat
```go
// pkg/kubelet/kubelet.go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {

    // ... 启动其他组件
    
    if kl.kubeClient != nil {
        // Introduce some small jittering to ensure that over time the requests won't start
        // accumulating at approximately the same time from the set of nodes due to priority and
        // fairness effect.
        // 开始NodeStatus状态同步循环
        go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
        // 进行一次快速状态更新
        go kl.fastStatusUpdateOnce()

        // start syncing lease
        // 开始Lease续约循环
        go kl.nodeLeaseController.Run(wait.NeverStop)
    }

    // ... 启动其他组件
}
```