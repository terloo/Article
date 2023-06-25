# HPA
HPA(Horizontal Pod Autoscaler)是k8s的一个资源对象，能够根据某些指标对在statefulset、replicaset、deployment等集合中的Pod数量进行横向动态伸缩，使应用服务具有一定的适应变化能力

## 特点
1. 因节点计算资源固定，当Pod完成调度后，动态调整计算资源变得较为困难，所以横向扩展具有更大的优势，HPA是扩展应用能力的第一选择
2. 多个冲突的HPA同时创建到同一个应用时，会有无法预期的行为，因此需要小心维护HPA规则
3. HPA依赖于metrics-server

## 清单
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  maxReplicas: 10
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```

## HPA支持的指标类型
即`spec.metrics.type`的可选值
1. Resource：k8s的内置指标类型，`resource.name`可以为cpu、memory，该指标的target可以是Utilization和AverageValue
2. Pods：自定义指标类型，`resource.name`为自定义指标名，该指标的target只能是AverageValue

## 扩缩容算法
期望副本数=ceil[当前副本数*(当前指标/期望指标)]  
如果计算出的扩缩容比例接近1.0(根据--horizontal-pod-autoscaler-tolerance参数配置全局容忍值，默认为0.1)，将会放弃本次扩缩容

## 滚动升级时扩缩容
HorizontalPodAutoscaler管理Deployment的replicas字段，而Deployment Controller负责设置下层ReplicaSet的replicas字段，以便确保在上线及后续的过程中副本个数合适

## 冷却/延迟支持
当使用HorizontalPodAutoscaler管理一组副本扩缩容时，有可能因为指标的动态变化造成副本数量的频繁变化，称为抖动(Thrashing)。  
--horizontal-pod-autoscaler-downscale-stabilization选项(默认值5m0s)可以设置**缩容**冷却时间窗口长度。HPA能记住过去建议的负载规模，并仅对此时间窗口内的最大规模执行操作。

## 扩缩容行为
应用在扩缩容时，每次扩缩容的实例数。即`spec.behavior`字段，可以指定一个或多个扩缩容策略。当指定多个策略时，默认选择允许更改最多实例数的策略
```yaml
behavior:
  scaleDown:     # 定义在缩容时的规则
    policies:
    - type: Pods
      value: 4
      periodSeconds: 60
    - type: Percent
      value: 10
      periodSeconds: 60
```

## 存在的问题
因弹性控制器的操作链路过长，基于指标的扩缩容有滞后效应，从应用超出负载阈值到HPA完成扩容之间的时间差包括
1. 应用指标数据超过阈值
2. HPA定期执行指标收集滞后的时间
3. HPA控制Deployment进行扩容的时间
4. Pod调度、运行时启动挂载存储和网络的时间
5. 应用服务启动到就绪的时间
很可能在突发流量出现时，还没完成弹性扩容，既有的服务实例已被流量击垮