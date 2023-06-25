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
StatefuleSet必须使用headlessService作为应用的Service
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

