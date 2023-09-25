# Ingress
Ingress是k8s中对集群的入口流量进行流量管理的资源对象

## 组件
1. Ingress资源对象：k8s的内置资源对象，配置信息
3. IngressController：监控Ingress资源对象的变化，实现Ingress资源对象所配置的规则。一般以Daemonset的形式部署，实现反向代理功能的服务

## Ingress访问过程
1. 外部流量负载均衡到某个节点上的IngressController中
2. IngressController根据集群中的Service配置，将流量代理到Service的某个Pod

## 资源对象
```yaml
apiVersion: network.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
  namespace: default
spec:
  ingressClassName: nginx # 使用哪个ingressClass
  # ingress将会从上到下对所有的rule进行匹配，并转发流量到第一个匹配到的规则
  rules:
  - host: demo.for.bar # 虚拟主机的FQDN，支持"*"前缀通配，不支持IP，不支持端口
    http:
      paths:
      - path: /hello # 必须以/开头，匹配HTTP的路径，支持正则表达式
        pathType: Prefix # 支持Exact、Prefix和ImplementationSpecific
        backend:
          service:
            name: demo-svc # 关联的后端Service资源对象名
            port: # 端口号或者端口名
              name: demo-port
              number: 80
  # rules未匹配时，使用的默认后端，数据类型与backend相同
  defaultBackend: {}
  # 指定需要工作在tls模式下的host
  tls:
    hosts: []
    secretName: demo-secret # 保存了tls证书的secret
```

## pathType
对于http请求的路径匹配，一共有三种匹配方案
1. Exact：精准匹配
2. Prefix：前缀匹配
3. ImplementationSpecific：由IngressClass进行匹配