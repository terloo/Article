# Endpoints
Endpoints是k8s中用于关联Service与Pod的对象

## 工作流程
1. 当Service的selectot不为空时，EndpointController会侦听服务创建对象，创建与Service同名的Endpoints对象
2. seletor能选中的所有PodIP会被配置到address属性中
   1. 如果此时seletor所对应的filter查询不到pod，则address列表为空
   2. 默认配置下，如此时Pod的状态为NotReady，则对应的Pod的IP只会出现在Endpoints的notReadyAddress中，意味着Pod还未就绪，不能向其转发流量
   3. 如果Service设置了PublishNotReadyAddress为true，则无论Pod是否就绪，都会被加入readAddress列表中

## 清单
```yaml
kind: Endpoints
metadata:
  creationTimestamp: "2022-11-12T09:40:57Z"
  name: mysql
  namespace: default
  resourceVersion: "58637590"
  uid: 650beabc-a83c-44ed-b8fb-e70b02bec468
subsets:
- addresses:
  - ip: 172.17.0.7
    nodeName: master
    targetRef:
      kind: Pod
      name: mysql-7876b687df-xsvm6
      namespace: default
      uid: 09e6577c-9af2-499a-b5cf-bab2f1b4dad6
  ports:    # 从Service中拷贝而来
  - port: 3306
    protocol: TCP
```

## EndpointSlice
在创建一个Endpoint对象之后，k8s会自动创建一个EndpointSlice对象用于分割Endpoint
1. 当某个Service对应的Pod特别多时，Endpoints对象就会因为保存了过多地址信息变得过于庞大
2. Pod状态变更引发Endpoints变更，Endpoints的变更会被推送至所有节点，从而导致持续占用过多网络带宽
3. 可以使用EndpointSlice对象，用于对后端地址过多的Endpoints进行切片，切片大小可以自定义

## 未指定selector的Service
当一个Service未指定seletor时，该Service将不会被EndpointController所管理，用户可以通过apiServer自行修改该Endpoints对象的address字段来为该Endpoints对象分配存储后端