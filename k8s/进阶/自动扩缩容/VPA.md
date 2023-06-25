# VPA
VPA(Vertical Pod Autoscaler)，即垂直Pod自动扩缩容。根据容器资源使用率自动设置CPU和memory的requests值，从而允许在节点上进行适当的调度，以便为Pod提供适当的资源。

## 意义
1. Pod资源用启所需，提升节点使用效率
2. 不必运行基准测试任务来决定CPU和内存请求的合适值
3. VPA可以随时调整CPU和内存的请求，无需认为操作，因此可以减少维护时间

## 组件
1. VerticalPodAutoscaler：k8s内置资源
2. VPA Recommender：监视所有Pod，不断为它们计算新的资源推荐，并将推荐资源值存储在VPA对象中。使用来自metrics-server获取Pod资源利用率和OOM事件
3. VPA Admission Controller：所有Pod创建请求都通过其管理资源
4. VPA Updater：负责Pod实时更新的组件。如果Pod在auto模式下使用VPA，则Updater可以决定使用推荐器资源对其进行更新
5. History Storage：使用来自apiServer的Pod资源利用率信息和OOM信息，并将其持久化(如Prometheus)