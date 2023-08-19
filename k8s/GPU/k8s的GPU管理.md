# k8s的GPU管理

## Pod使用gpu资源
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - image: "nvidia/cuda-vector-add:v0.1"
    name: pod-gpu
    resources:
      # 在资源配置中使用GPU资源
      limit:
        nvidia.com/gpu: 1
```

## 管理节点GPU资源
1. Extended Resources：通过自定义资源进行扩展，直接根据实际情况手动修改节点的自定义资源
2. Device Plugin Framework：部署GPU厂商对应的设备插件，以外置的方式自动进行GPU设备的生命周期管理

## DevicePluginFramework
由GPU厂商实现的GRPC服务，一般使用Deamonset的形式部署到所有节点上
1. Registry：向Kubelet注册本插件，以便Kubelet对Plugin接口进行调用
2. ListAndWatch：负责将节点上的GPU设备状态及其状态上报到Kubelet，由Kubelet更新并维护节点状态。
3. Allocate：在容器被调度到节点/从节点上回收时被调用，Kubelet会将需要对设备进行的操作请求到DevicePlugin，DevicePlugin会返回被分配的设备列表/数据集/环境变量或者回收指定的设备

## 缺陷
1. 对设备的调度主要发生在Kubelet层面，无法在全局层面进行调度
2. 资源上报的信息有限(只有数量)导致调度精细度不够
3. 调度策略简单，无法应对复杂环境