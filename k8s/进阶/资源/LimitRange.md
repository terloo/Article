# LimitRange

## 资源定义
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mylimit
spec:
  limits:
  - type: Pod
    max:
      cpu: "4"
      memory: 2Gi
    min:
      cpu: "200m"
      memory: 6Mi
    maxLimitRequestRatio:
      cpu: 3
      memory: 2
  - type: Container
    default:
      cpu: 300m
      memory: 200Mi
    defaultRequest:
      cpu: 200m
      memory: 100Mi
    max:
      cpu: "2"
      memory: 1Gi
    min:
      cpu: 100m
      memory: 3Mi
    maxLimitRequestRatio:
      cpu: 5
      memory: 4
```

## 字段解释
1. Container的Min是Pod中所有容器的Requests值下限；Container的Max是Pod中所有容器的Limits值的上限。
2. Container的DefaultRequest是Pod中所有未指定Requests值的容器的默认Requests值，Container的DefaultLimit(字段名为default)是Pod中所有未指定Limits值的容器的默认Limits值
3. 以上四个参数，必须满足Max >= DefaultLimit >= DefaultRequest >= Min
4. Pod的Min是Pod中所有容器的Requests值总和的下限，Pod的Max是Pod中所有容器Limits值总和上限
5. Container的MaxLimitRequestsRatio限制了Pod中所有容器的Limits值和Requests值的比例上限；而Pod的MaxLimitRequestsRatio限制了Pod中所有容器的Limits值和Requests值总和的比例上限

## Describe
```
Namespace:  default
Type        Resource  Min   Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---   ---  ---------------  -------------  -----------------------
Pod         memory    6Mi   2Gi  -                -              2
Pod         cpu       200m  4    -                -              3
Container   cpu       100m  2    200m             300m           5
Container   memory    3Mi   1Gi  100Mi            200Mi          4
```

## 注意事项
1. 不论是CPU还是内存，在LimitRange中，Pod和Container都可以设置Min、Max和MaxLimitRequestsRatio参数，Container还可以设置DefaultRequest和DefaultLimit参数。
2. 如果设置了Container的Max，那么对于该类资源而言，整个集群中所有的容器都必须设置Limits，如果没有设置，将采用DefaultLimits(字段名为default)，如果DefautlLimits也未设置，则Pod无法创建。
3. 命名空间中的LimitRange只会在Pod创建或者更新时执行检查。如果手动修改LimitRange，将不会检查或设置现有的Pod设置
