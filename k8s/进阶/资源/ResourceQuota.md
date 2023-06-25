# ResourceQuota
ResourceQuota(资源配额)限制命名空间下的资源使用，可以限制命名空间中某种类型的对象的总数量限制，也可以限制命名空间中Pod可以使用的计算资源和存储资源上限

## 开启资源配额选型
资源配额可以在api-server中通过向参数--admission-control添加ResourceQuotas选项进行开启。  
如果一个命名空间中存在ResourceQuotas对象，那么对于该命名空间而言，资源配额就是开启的。一个命名空间可以有多个资源配额

## 资源定义
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-ns-resource
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 100Gi
    limits.cpu: "200"
    limits.memory: 200Gi
```

## 资源配额配置

### 计算型资源配额
| 资源名称        | 说明                                                 |
| --------------- | ---------------------------------------------------- |
| Cpu             | 所有非终止状态的Pod，CPU Requests的总和不能超过该值  |
| Memory          | 所有非终止状态的Pod，内存 Requests的总和不能超过该值 |
| limits.cpu      | 所有非终止状态的Pod，CPU Limits的总和不能超过该值    |
| limits.memory   | 所有非终止状态的Pod，内存 Limits的总和不能超过该值   |
| reqeusts.cpu    | 所有非终止状态的Pod，CPU Requests的总和不能超过该值  |
| requests.memory | 所有非终止状态的Pod，内存 Requests的总和不能超过该值 |

### 存储型资源配额：
| 资源名称                                                                | 说明                                                            |
| ----------------------------------------------------------------------- | --------------------------------------------------------------- |
| requests.storage                                                        | 所有PVC，存储请求总量不能超过该值                               |
| persistentvolumeclaims                                                  | 在命名空间中能存在的持久卷的总数上限                            |
| <storage-class-name>.storageclass.storage.k8s.io/reqeusts.storage       | 所有与<storage-class-name>相关的PVC请求的存储总量都不能超过该值 |
| <storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims | 所有与<storage-class-name>相关的PVC的总数都不超过该值           |
| ephemeral-storage、requests.ephemeral-storage、limits.ephemeral-storage | 本地临时存储(ephemeral-storage)的总量限制                       |

### 对象数量配额
1. count/<resources>.<groups>：用于非核心组的资源
2. count/<resource>：用于核心组的资源