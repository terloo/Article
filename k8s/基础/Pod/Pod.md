# Pod
Pod是一组容器的集合，是k8s进行容器调度的最小单位。Pod中的容器共享网络，可以通过容器卷共享存储。

## 基础定义
```yaml
apiVersion: v1
kind: pod
metadata: 
  name: myapp
  namespace: default
spec:
  containers:
  - name: app
    image: nginx
```

## 必须存在的属性
| 字段名                  | 字段类型 | 说明                                                            |
| ----------------------- | -------- | --------------------------------------------------------------- |
| apiVersion              | string   | 指k8s api的版本，可以使用`kubectl api-versions`查询，一般为`v1` |
| kind                    | string   | yaml文件定义的资源类型和角色，这里为Pod                         |
| metadata                | Obj      | 元数据对象，固定值就写metadata                                  |
| metadata.name           | string   | 元数据对象的名字，由用户进行指定                                |
| metadata.namespace      | string   | 元数据对象的命名空间，由用户指定，其实可以不写，默认default     |
| spec                    | Obj      | 详细定义对象                                                    |
| sepc.containers[]       | list     | Pod中容器定义列表                                               |
| spec.containers[].name  | string   | 定义容器的名字                                                  |
| spec.containers[].image | string   | 定义要用到的镜像名                                              |