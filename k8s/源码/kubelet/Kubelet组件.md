# Kubelet组件

## PLEG
即`Pod Lifecycle Event Generator`，Pod生命周期(ContainerStarted,ContainerDied,ContainerRemoved,ContainerChanged)事件的生成器  
其内部维护着Pod中所有容器的状态，定期通过`ContainerRuntime`获取Pod中所有Container状态于缓存中状态比较，生成如上的事件。  
将事件写入其维护的channel和同步到container.Cache中

## PodConfig
一个配置多路复用器，将多个Pod配置源合并成单一的结构一致的变更通知，然后按顺序将结果写入channel中

## ProbeManager
包含了三种Probe，缓存probe的结果，并将结果写入channel中
- LivenessManager
- ReadinessManager
- StartupManager

## syncLoop
调用接收来自PodConfig、定时同步、PLEG、ProbManager的channel的事件，将Pod同步到期望状态

## PodWorkers
处理Pod相关的事件的多个Worker集合，由syncLoop调用。其核心方法`managePodLoop()`中调用了`kubelet.syncPod()`完成Pod的同步

## PodManager
PodManager缓存着该节点拥有的所有的Pod的缓存，当Pod产生主动或被动性的变动时，kubelet会更新其中的缓存。  
PodManager也维护着`static pod`到`mirror pod`之间的映射关系

## StatusManager
StatusManager缓存着该节点所有Pod的状态，并将状态同步到api-server。在Pod被删除且资源回收完毕后，移除Pod

## container.Cache
container.Cache缓存着该节点所有容器的当前状态，被PLEG所更新，用于其他组件获取该节点上Pod的真实状态

## StatsProvider

## ContainerRuntime

## PodAdmitHandlers
处理Pod admission过程中调用的一系列Handler。比如eviction handler(节点有压力时，不会驱逐QoS设置为BestEffort的Pod)等

## OOMWatcher
从容器中获取OOM日志，将其封装成事件并记录

## VolumeManager
根据Pod的配置，确定是否需要附加/挂载/卸载/分离哪些卷并执行操作

## CertificateManager
证书轮换

## EvictionManager
监控Node节点资源的占用情况，根据驱逐规则驱逐Pod释放资源，缓解节点压力

## PluginManager

## CSI
Container Storage Interface，有存储厂商实现的存储驱动

## Device Plugin
设备管理器插件，是由kubelet提供的一个设备插件框架，可以使用此框架将系统的硬件资源发布到Kubelet  
供应商可以实现设备插件，由用户手动部署为DaemonSet，而不须修改k8s的代码。  
目标设备包括GPU、高性能NIC、FPGA、InfiniBand适配器以及其他类似的可能需要特定于供应商初始化和设置的计算资源
