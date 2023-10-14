# BGPPeer
BGPPeer是calico中用于申明一个或多个BGP对等体的CRD，可以指定一个多个或所有现存node来连接该对等体。  
mesh模式下，所有的Node都会被视为BGPPeer，可以先设置BGPPeer再关闭mesh模式防止网络中断

## Sample
```yaml
# 申明一个对等体，可以是物理路由器的ip，所有的Node都会对其进行连接
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: some.name
spec:
  peerIP: 192.168.1.1
  asNumber: 63400

---
# 利用标签选择器，申明多个Node为BGPPeer，在非mesh的情况下，相当于缩小了mesh的范围
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: some.name
spec:
  peerSelector: key == 'value'

---
# 利用标签选择器，使所有节点都连接route-reflector
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: some.name
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
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