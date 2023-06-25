# volumes
volumes是Pod中用于定义Pod所使用的所有容器卷的字段

## 基础
| 字段名                 | 字段类型 | 说明                                                            |
| ---------------------- | -------- | --------------------------------------------------------------- |
| sepc.containers[]      | list     |                                                                 |
| spec.containers[].name | string   | 定义容器的名字，会被spec.containers[].volumeMounts[].name所引用 |

## 类型
1. 本地存储
   1. emptyDir：emptyDir在Pod创建过程中创建的临时目录，在Pod结束后将会直接删除，在宿主机上的路径为`/var/lib/kubelet/pods/<podUID>/volume/kubernetes.io~empty-dir/<volumeName>`
   2. hostPath：宿主机上的路径，Pod结束后依然存在
2. 网络存储
   1. intree：由k8s进行实现的网络存储
   2. out-of-tree：由第三方插件进行实现的网络存储
3. Projected Volume
   1. Secret
   2. ConfigMap
   3. DownwardAPI
   4. ServiceAccountToken
4. PV与PVC体系
