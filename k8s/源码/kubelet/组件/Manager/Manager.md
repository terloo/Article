# Manager
Kubelet中使用的Manager的集合
1. baseManger：分为cacheBasedManager和watchBasedManager
2. ConfigMapManager：管理Pod的ConfigMap，由baseManager实现
3. SecretManager：管理Pod的Secret，由baseManager实现
4. PodManager：缓存Pod的期望配置(Spec字段)，管理static pod到mirror pod的映射，并将mirror pod同步到api-server
5. StatusManger：缓存Pod的当前状态(Status字段)，并将当前Pod状态同步到api-server