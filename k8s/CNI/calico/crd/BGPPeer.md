# BGPPeer
BGPPeer是calico中用于申明一个或多个BGP对等体的CRD，可以指定一个多个或所有现存node来连接该对等体

## Sample
```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: some.name
spec:
  peerIP: 192.168.1.1
  asNumber: 63400
```

## 字段
| 字段名              | 字段类型       | 说明                                                                                 |
| ------------------- | -------------- | ------------------------------------------------------------------------------------ |
| peerIP              | IP             | 现存BGP对等体的IP地址，该对等体不仅可以是Node，也可以是路由器等运行BGP协议的网络设备 |
| asNumber            | int            | 现存BGP对等体的AS号                                                                  |
| peerSelector        | selector表达式 | 如果是配置Node为BGP对等体，也可以使用选择器批量指定，与peerID和asNumber字段互斥      |
| node                | String         | 需要连接BGP对等体的节点                                                              |
| nodeSelector        | selector表达式 | 需要连接BGP对等体的多个节点，如果该字段和node同时为空，代表所有节点                  |
| keepOriginalNextHop | boolean        | 在EBGP时，是否保持原有下一跳地址                                                     |
| maxRestartTime      | String         | 连接保活超时时间，如10s,2m，默认120s                                                 |