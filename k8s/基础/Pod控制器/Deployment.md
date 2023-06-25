# Deployment

## 资源清单示例(deployment举例)
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
        app: nginx
  revisionHistoryLimit: 3  # 保留多少个历史版本，设置为0时不保留历史版本
  replicas: 3
  template:
  # 以下开始其实嵌套的是Pod清单中的值，只是不用设置pod名，因为会自动生成
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-pod
        image: nginx
```

## 升级策略
| 字段                                       | 值                              | 说明                                                                                    |
| ------------------------------------------ | ------------------------------- | --------------------------------------------------------------------------------------- |
| spec.strategy.type                         | string(Recreate或RollingUpdate) | 升级策略，Recreate在所有Pod删除后再新建，RollingUpdate滚动升级                          |
| spec.strategy.rollingUpdate                | Object                          | 在type为RollingUpdate时，配置相关细节                                                   |
| spec.strategy.rollingUpdate.maxSurge       | string(可以为数字或者百分比)    | 表示在滚动过程中，允许同时部署的新Pod的数量(向上取整)，默认为25%。需要考虑ResourceQuota |
| spec.strategy.rollingUpdate.maxUnavailable | string(可以为数字或者百分比)    | 表示在滚动过程中，允许同时终止的旧Pod的数量(向上取整)，默认为25%                        |

## 相关命令
1. 部署`kubectl apply -f deployment.yaml --record  # 使用record能记录下这次命令，可以方便查看每次revision的变化，方便回滚`
2. 滚动更新`kubectl set image deployment deployment_name container_name=new_image`
3. 扩容缩容`kubectl scale deployement deployment_name --replicas n # n指定扩容缩容数量`
4. HPA(如果有)`kubectl autoscale deployment deployment_name --min=10 --max=15 --cpu-precent=80`设置如果cpu利用率上了80则扩容最多到15
5. 版本回退
```shell
kubectl rollout restart deployment deployment_name # 重启deployment下所有Pod
kubectl rollout status deployment deployment_name # 查看更新状态
kubectl rollout history deployment deployment_name # 查看历史版本，版本号最大的是当前版本
kubectl rollout undo deployment deployment_name  # 回滚到上一版本(最大版本号 - 1)
kubectl rollout undo deployment deployment_name --to-revision=n  # 回滚到第n个版本
kubectl rollout pause deployment deployment_name  # 暂停deployment的更新，一般用于灰度发布
kubectl rollout resume deployment deployment_name  # 恢复deployment的更新
```
