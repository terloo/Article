# VirtualService
配置一系列的匹配规则用于流量匹配，匹配成功的流量将会根据route中的destination来路由。对应xDS中的RDS。

## 示例
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: mybackend-vs
spec:
  hosts:
  - backend-svc # 一个k8s service，istio将其完全代理
  http:
  - match:
    - headers:  # 匹配请求头有foo:bar的请求
        foo:
          exact: bar
    route:
    - destination:
        host: nginx-svc  # 代理到一个k8s service
        port:
          number: 80
  - route:     # 没有匹配条件，放在匹配链最末尾的默认路由路径
    - destination:
        host: nginx-svc  # 代理到一个k8s service
        port:
          number: 80
      weight: 25
    - destination:
        host: tomcat-svc # 代理到另一个k8s service
        port:
          number: 80
      weight: 75
```

## 字段说明
| 字段名                                 | 类型        | 说明                                           |
| -------------------------------------- | ----------- | ---------------------------------------------- |
| hosts                                  | stringArray | 需要匹配的路由的目的地址                       |
| http                                   | ObjectArray | 匹配http类型的流量                             |
| tcp                                    | ObjectArray | 用于匹配tcp的流量                              |
| tsl                                    | ObjectArray | 用于匹配tsl的流量                              |
| http[].match                           | ObjectArray | 匹配规则                                       |
| http[].match.headers.xxxx              | Object      | 用于描述请求头的匹配条件                       |
| http[].match.headers.xxxx.exact        | string      | 精确匹配                                       |
| http[].match.headers.xxxx.prefix       | string      | 前缀匹配                                       |
| http[].match.headers.xxxx.regex        | string      | 正则匹配                                       |
| http[].rewrite.uri                     | string      | 重写uri                                        |
| http[].route                           | ObjectArray | 路由目的地，多个目的地可以使用weight来指定权重 |
| http[].route[].weight                  | int         | 权重                                           |
| http[].route[].destination             | Object      | 目标地                                         |
| http[].route[].destination.host        | string      | 目标ip或域名                                   |
| http[].route[].destination.port.number | int         | 目标端口                                       |

## 注意
1. 如果多个VirturalService的Host地址有重复，采用创建时间较小的那一个