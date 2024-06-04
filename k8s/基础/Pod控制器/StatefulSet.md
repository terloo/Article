# StatefulSet

## 特点
1. Pod一致性：包含启停顺序、网络一致性。此一致性与Pod相关，与Pod被调度到哪个节点无关
2. 稳定的次序：对于N个副本的StatefulSet，每个Pod都在[0，N)的范围内分配一个数字序号，且是唯一的；
3. 稳定的网络：Pod的hostname模式为(statefulset名称)−(序号)；
4. 稳定的存储：通过VolumeClaimTemplate为每个Pod创建一个PV。删除、减少副本，不会删除相关的卷。

## 资源清单
```yaml
apiVersion: apps/v1
Kind: StatefulSet
metadata: 
  name: nginx-sts
  label:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx-svc" # 必填
  template:
    metadata:
      name: nginx
      label:
        app: nginx
    containers:
    - name: nginx
      image: nginx
      ports:
      - name: http
        containerPort: 80
      volumeMounts:
      - name: www-storage
        path: /usr/share/nginx/html
  volumeClaimTemplates:    # PVC模板
  - metadata:
      name: www-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Mi
```
| 字段                      | 默认值 | 说明                                                                                  |
| ------------------------- | ------ | ------------------------------------------------------------------------------------- |
| spec.serviceName          | 无     | 该StatefulSet使用的HeadlessService的名字，必须在StatefulSet之前创建                   |
| spec.template             | 无     | 该StatefulSet使用的Pod模板，StatefulSet将会自动创建PVC，并将PVC的回收策略设置为retain |
| spec.volumeClaimTemplates | 无     | 该StatefulSet使用的PVC模板                                                            |

## 无头服务
StatefulSet必须使用headlessService作为应用的Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-sts-headlessservice
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
  - port: 80
```

## Pod的版本
与Deployment通过ReplicaSet来管理Pod版本不同，StatefulSet通过ControllerRevision来管理Pod的版本。  
StatefulSet的Pod模板被更新后，StatefulSet会创建一个新的ControllerRevision来标识新的Pod模板。  
并由序号从大到小的顺序依次更新所有的Pod

## 部署和扩缩容顺序保证
1. 对于包含N个副本的StatefulSet，当部署Pod时，它们是依次创建的，顺序为`0 ~ N-1`
2. 当删除Pod时，它们是逆序终止的，顺序为`N-1 ~ 0`
3. 在将扩缩容应用到Pod之前，它前面的所有Pod必须是ReadyRunning或完全停止状态
4. 在一个Pod终止之前，所有的继任者必须完全关闭
5. 顺序保证可以通过`.spec.podManagementPolicy`进行管理
   1. OrderedReady：默认策略，最严格的顺序保证
   2. Parallel：在扩缩容时，Pod可以并行的扩缩容。**更新时仍需保证顺序**

## 更新策略updateStrategy
通过`.spec.updateStrategy`可以控制statefulSet的更新策略
1. `.spec.updateStrategy.type`为`OnDelete`时，用户必须手动删除旧Pod以便让控制器创建新Pod
2. `.spec.updateStrategy.type`为`RollingUpdate`时，用户通过配置`.spec.updateStrategy.rollingUpdate`来控制自动的执行的滚动更新，这是默认的策略

## 滚动更新RollingUpdate
`.spec.updateStrategy.type`为`RollingUpdate`时，控制器将按Pod序号`N-1 ~ 0`的顺序依次更新Pod。如果设置了`.spec.minReadySeconds`，在Pod进入Running且Ready状态时还会额外等待一定时间秒数
1. partitions：通过设置`.spec.updateStrategy.rollingUpdate.partition`字段实现分区滚动更新。当StatefulSet被更新时，所有序号大于等于该值的Pod都会被更新，序号小于该值的Pod不会被更新，即使强行删除也会重建为旧版本。若该值大于最大序号，则更新操作不会产生实质效果

## 回滚
由于StatefulSet的滚动更新需要等待Pod进行Running且Ready状态才会继续推进下一个Pod。如果该Pod产生了错误，即使回滚了StatefulSet模板，控制器也不会进行处理，此时需要人工干预删除该错误Pod