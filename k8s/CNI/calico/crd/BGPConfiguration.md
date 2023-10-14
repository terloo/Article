# BGPConfiguration
BGPConfiguration是用于配置全局或节点BGP相关参数的资源对象，该资源对象有可以有多个

## 特殊值
1. 名为`default`：代表着全局默认值
2. 名为`node.<nodeName>`：代表着节点覆盖值，会应用到<nodeName>的节点上。删除节点时，对应的BGPConfiguration也会被删除。通过这个方法，只能覆盖`prefixAdvertisements`、`listenPort`和`logSeverityScreen`字段的值

## sample
```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  nodeMeshMaxRestartTime: 120s
  asNumber: 63400
  serviceClusterIPs:
    - cidr: 10.96.0.0/12
  serviceExternalIPs:
    - cidr: 104.244.42.129/32
    - cidr: 172.217.3.0/24
  listenPort: 178
  bindMode: NodeIP
  communities:
    - name: bgp-large-community
      value: 63400:300:100
  prefixAdvertisements:
    - cidr: 172.218.4.0/26
      communities:
        - bgp-large-community
        - 63400:120
```

## 字段
| 字段名                 | 字段类型                 | 说明                                                                    | 默认值 |
| ---------------------- | ------------------------ | ----------------------------------------------------------------------- | ------ |
| logSeverityScreen      | String                   | Debug, Info, Warning, Error, Fatal日志等级                              | Info   |
| listenPort             | int                      | BGP协议监听端口                                                         | 179    |
| nodeToNodeMeshEnabled  | boolean                  | 只在default中起作用，是否开启mesh模式                                   | true   |
| asNumber               | int                      | 只在default中起作用，默认AS号                                           | 64512  |
| serviceClusterIPs      | CIDRList                 | 只在default中起作用，需要通过BGP宣告的ClusterIP范围                     | empty  |
| serviceExternalIPs     | CIDRList                 | 只在default中起作用，需要通过BGP宣告的ExternalIP范围                    | empty  |
| serviceLoadBalancerIPs | CIDRList                 | 只在default中起作用，需要通过BGP宣告的LoadBalancerIP范围                | empty  |
| bindMode               | String                   | BGP监听的IP地址，None代表0.0.0.0，NodeIP，修改后需要重启calico-node生效 | None   |
| nodeMeshPassword       | BGPPassword              | mesh模式下，使用的密码                                                  | nil    |
| nodeMeshMaxRestartTime | time                     | mesh模式下，BGP重启时间                                                 | 120s   |
| ignoredInterfaces      | StringList               | 不读取路由的设备，可以使用通配符*                                       | nil    |
| communities            | communitiesList          | BGP团体名称                                                             |        |
| prefixAdvertisements   | prefixAdvertisementsList | BGP团体名称，配置细化到前缀级别                                         |        |