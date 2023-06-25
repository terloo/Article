# dnsPolicy
dnsPolicy是k8sPod中一个属性`spec.dnsPolicy`，用于定义Pod的DNS解析策略

## ClusterFirst
配置为`ClusterFirst`时，将会采用集群中的CoreDNS进行解析

## Default
配置为`Default`时，将会采用宿主机的域名解析配置进行域名解析

## None
配置为`None`时，可以手动指定DNS服务器地址
```yaml
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
    - 114.114.114.114
    - 8.8.8.8
    searchs:
    - xx.ns1.svc.cluster.local
    - ns1.svc.cluster.local
    options:
    - name: ndots
      values: "s"
```