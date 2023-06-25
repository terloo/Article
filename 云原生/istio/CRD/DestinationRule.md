# DestinationRule
DestinationRule可以按label将一组服务划分为不同的子集，并根据子集的不同，进行精细化的流量管理策略。

## 示例
```yml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-dr
spec:
  host: backend-svc  # 一组服务，在k8s中，通过EntryPoint来划分服务
  trafficPolicy:     # 对该组服务的全局流量策略
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: old-backend  # 使用label，将pod划分为ds中不同的subset，避免了k8s service的使用
    labels:
      app: tomcat-pod
  - name: new-backend
    labels:
      app: nginx-pod
```

## 字段说明
| 字段                  | 类型        | 说明                                               |
| --------------------- | ----------- | -------------------------------------------------- |
| host                  | string      | dr的host，到此host的流量代表着访问该dr             |
| subsets               | ObjectArray | dr的所有子集，可以在vs中控制对这些子集的访问策略   |
| subsets.name          | string      | 该子集的名字                                       |
| subsets.lables        | Object      | 子集的匹配条件                                     |
| trafficPolicy         | Object      | 子集内所有服务实例的默认路由策略                   |
| subsets.trafficPolicy | Object      | 该子集内所有服务实例的路由策略，会覆盖默认路由策略 |