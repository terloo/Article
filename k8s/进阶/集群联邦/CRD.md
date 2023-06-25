# CRD
成员集群是联邦的基本管理单元，所有待管理集群均须注册到集群联邦  
集群联邦V2提供了统一的工具集(kubefedctl)，允许用户对单个对象动态的创建联邦对象。动态对象的生成基于CRD

|          | k8s集群    | k8s集群联邦         |
| -------- | ---------- | ------------------- |
| 计算资源 | Node       | Cluster             |
| 部署     | Deployment | FederatedDeployment |
| 配置     | ConfigMap  | FederatedConfigMap  |
| 密钥     | Secret     | FederatedSecret     |
| 其他对象 | XXX        | FederatedXXX        |