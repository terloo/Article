# k8s架构

1. 集群节点角色功能
   1. Master Node 控制节点
      - 集群的控制节点，对集群进行调度管理，接受集群外用户对集群的操作请求；
      - Master Node由ApiServer、Scheduler、Cluster State Store（etcd数据库）和Controller Manger Server所组成。
   2. Worker Node 工作节点
      - 集群工作节点，运行用户业务应用容器。
      - Worker Node包含Kubelet、Kube proxy和Container Runtime（一般是用docker）
      - 最好>=3的奇数个

2. Master Node
   1. ApiServer
      提供集群与外界交互的api。所有服务访问的统一入口。
   2. scheduler
      负责将pod调度到相应的Node中。
   3. Controller Manager Server
      维持副本的期望数目。
   4. ETCD
      键值对数据库，存储k8s集群的所有（需要持久化的）重要信息

3. Worker Node
   1. Kubelet
      向Master注册自己，定时向Master汇报自身和Pod的情况。并负责与CRI（container runtime interface）进行交互创建容器，维持Pod的生命周期。
   2. Kube proxy
      负责将规则写入iptables、IPVS实现服务映射访问。
   3. Container
      容器引擎，一般使用Docker

4. 一些常用插件
   1. CoreDNS
      可以为集群中的svc创建一个域名ip的对应关系解析。
   2. DashBoard
      给k8s集群提供一个BS结构的访问体系
   3. Ingress Controller
      官方只能实现四层代理，Ingress可以实现七层代理。
   4. Federation
      提供一个可以跨集群中心多k8s管理
   5. Prometheus
      提供一个k8s集群的监控能力
   6. EFK
      提供k8s集群日志统一分析介入平台

5. 节点需要的组件列表
   | Maste节点                      | Node节点                       | 说明                          |
   | ------------------------------ | ------------------------------ | ----------------------------- |
   | etcd                           |                                | 保存集群配置                  |
   | kube-apiserver                 |                                | 提供集群对外控制接口          |
   | kubectl                        |                                | 方便用户调用api server        |
   | kube-controller-manager        |                                | 控制Pod的生命周期             |
   | kube-scheduler                 |                                | 将Pod调度到Node上             |
   |                                | kube-proxy                     | 将集群外部网络请求代理到Pod上 |
   |                                | kubelet                        | 负责调度docker                |
   |                                | docker                         | 容器引擎                      |
   |                                | other apps                     |                               |
   | Contro plane(如calico，fannel) | Contro plane(如calico，fannel) | 网络插件                      |
