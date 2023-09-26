# NetworkPolicy
通过定义NetworkPolicy对象，可以配置Pod的进出流量访问规则，**需要csi支持**

## sample
NetworkPolicy是命名空间级别的资源对象
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-netpol
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: dev
      podSelector:
        matchLabels:
          app: busybox
    ports:
    - port: 80
```

## 字段
| 字段名                    | 字段类型    | 说明                                                                                                                                  |
| ------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| podSelector               | selector    | 需要应用此规则的Pod，如果没配置，则代表该命名空间下所有Pod                                                                            |
| policyTypes               | StringArray | 配置启用Ingress或者Egress或两者都有。如果未指定，则根据ingress和egress字段进行推断                                                    |
| ingress                   | ObjectArray | 入站规则，白名单。入栈流量同时满足from和ports中任一规则即视为满足该ingress规则，满足任一ingress规则即可入栈。空代表不允许任何入栈流量 |
| ingress.from              | ObjectArray | 匹配来源，逻辑或关系                                                                                                                  |
| ingress.ipBlock           | Object      | 来源通过ip块进行判断                                                                                                                  |
| ingress.ipBlock.cidr      | String      | ip块                                                                                                                                  |
| ingress.ipBlock.except    | StringArray | 从cidr字段中排除此ip                                                                                                                  |
| ingress.namespaceSelector | Object      | 来源通过命名空间的标签选择器判断                                                                                                      |
| ingress.podSelector       | Object      | 来源通过Pod的标签选择器判断                                                                                                           |
| ingress.ports             | ObjectArray | 匹配端口和协议                                                                                                                        |
| ingress.ports.port        | String      | 判断端口                                                                                                                              |
| ingress.ports.endPort     | int         | 如果指定该字段，则端口范围从port到endPort                                                                                             |
| ingress.ports.protocol    | String      | 判断协议，默认TCP                                                                                                                     |
| egress                    | ObjectArray | 出站规则，白名单。同ingress                                                              |
| egress.to                 | ObjectArray | 匹配目的地，逻辑或关系。同ingress.from                                                                                                |
| egress.ports              | ObjectArray | 匹配端口和协议，同ingress.ports                                                                                                       |