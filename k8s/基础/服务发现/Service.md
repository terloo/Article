# Service资源清单

## 定义
Service定义了这样一种抽象：一个Pod的逻辑分组，一种可以访问它们的策略——通常被称为微服务。Service通常使用lableSelector来访问Pod。  
Service只能提供4层负载均衡（ip加端口），添加Ingress方案可以支持7层负载均衡。

## 类型
1. ClusterIp：默认类型，为service自动分配一个仅Cluster内部可以访问的虚拟IP。可以被集群中的Node和Pod都可以通过这个ip访问被service选择的pod。
2. NodePort：Service在**每个**Node机器上开一个相同端口，这样就外部就可以通过NodeIP:NodePort来访问该service。
3. LoadBalancer：在NodePort的基础上，借助cloud provider创建一个外部负载均衡器，并将请求转发到NodeIP:NodePort。（需要云服务商提供支撑）
4. ExternalName：把集群外部的服务引入到集群内部并给予外部地址一个域名或主机名，可以在集群内部直接联通。没有任何类型代理被创建。

## VIP和Service代理
在k8s集群中，每个Node都运行了一个kube-proxy的进程。kube-proxy负责为Service提供一种VIP（虚拟IP）的形式，而不是EXternalName的形式。1.14版本开始默认使用ipvs代理。
ipvs:
kube-proxy会监视Kubernets Service对象和Endpoints，调用netlink接口以相应的创建ipvs规则并定期与Kubernets Service对象和Endpoints对象同步ipvs规则，以确保ipvs状态与期望一致。访问服务时，流量将被重定向到其中一个后端Pod。
与iptables类似，ipvs基于netfilter的hook功能，但使用hash表作为底层数据结构并在内核空间中工作。这意味着ipvs可以更快的重定向流量，并且在同步代理规则时具有更好的性能。此外，ipvs为负载均衡提供了更多选项。包括：
1. rr：轮询调度
2. lc：最小连接数
3. dh：目标哈希
4. sh：源哈希
5. sed：最短期望延迟
6. nq：不排队调度
> ipvs假定在运行kube-proxy之前节点上都已经安装了IPVS模块。如果未安装，则kube-proxy将使用iptables代理模式。

## ClusterIp
ClusterIp主要在每一个节点上使用ipvs(或iptables)，将发向clusterip的每一个请求转发到kube-proxy中。然后kube-proxy自己内部实现负载均衡的方法，并且可以查询到这个service下对应的pod的地址和端口，进而把数据发给对应的pod的地址和端口。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nginx
spec:
  type: ClusterIp
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 3300
    targetPort: 80
```

## Headless Service(特殊的ClusterIP)
Headless Service并不会分配Cluster IP，kube-proxy不会处理他们，而且平台也不会为他们做负载均衡。用户可以通过域名直接解析到Pod的ip上
```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec: 
  selector: 
    app: nginx
  clusterIP: "None" # clusterIP为"None"时，这个Service为Headless Service
  ports:
  - port: 3301
    targetPort: 80
    name: headless
```

## NodePort
NodePort在ClusterIP基础上可以在**所有**节点上暴露一个端口，将该端口的流量代理到kube-proxy，kube-proxy再代理至容器内部服务，外部服务可以通过物理机IP加端口的形式访问至集群中的服务并实现Pod层面的负载均衡。如果不指定对外端口将会随机产生一个端口，默认端口范围为30000-32767。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodePort
spec:
  type: NodePort
  selector:
    app:nginx
  ports:
  - name: prot1
    port: 3303   #  ClusterIP端口
    targetPort: 80  # 映射到的pod内部端口
    nodePort: 32222  # 物理机开启的对外端口，所有物理机都会开启
```

## LoadBalancer
LoadBalancer是NodePort的扩展，在创建LoadBalancer时，第三方组件会给LoadBalancer分配一个集群外ip，集群外流量访问此ip时，第三方组件会将流量负载均衡到不同节点的NodePort上。

## ExternalName
这种类型的Service通过返回CNAME和它的值，可以将服务映射到externalName字段的内容(例如service-nginx.)。
ExternalName Service是Service的特例，它没有selector，也没有定义任何的端口和EndPoint。对于运行在集群外部的服务，它可以通过返回该外部服务的别名这种方式来提供服务。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: baidu
spec:
  type: ExternalName
  externalName: www.baidu.com
```

## Topology
为Service配置TopologyKeys属性后，Service的负载均衡将会根据TopologyKeys列表的优先级来确定流量路由的Endpoint

## Ingress(Nginx实现)
ingerss使用nginx可以实现7层代理，原理相当于在集群中再部署一个Nginx来实现域名的反向代理到Service(一般为ClusterIP)，可以避免Service过多时需要使用NodePort打开过多的物理机端口的情况，也可以处理https协议连接。

### http代理
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingerss
spec:
  rules:
  - host: www.csj.com  # 所有访问该域名的请求将会使用下列规则
    http: # http代理
      paths:  
      - path: /  # 所有访问该域名/目录下的请求
        backend:
          serviceName: nginx-service  # 将会被代理到该service的指定端口
          servicePort: 3300
```

### https代理
```yaml
apiVersion: extensions/v1beta1
kind: Ingerss
metadata:
  name: nginx-ingress
spec:
  tls:
  - hosts:
    - www.csj.com  # 该列表下所有的域名都将使用以下指定的证书
    secretName: tls-zhengshu  # 指定一个secret对象来引入证书
  rules:
  - host: www.csj.com
    paths:
    - path: /
      backend:
        serviceName: nginx-service
        serviceProt: 3300
```

### annotations中的一些配置选项
| 名称                                      | 值      | 描述                                                         |
| ----------------------------------------- | ------- | ------------------------------------------------------------ |
| nginx.ingress.kubernets.io/rewrite-target | str     | 必须重定向流量的URI                                          |
| nginx.ingress.kubernets.io/ssl-redirect   | boolean | 指示部分位置是否仅可访问ssl（当ingress包含证书时默认为True） |
| nginx.ingress.kubernets.io/force-direct   | boolean | 即使ingress未启用TLS，也强制重定向至https                    |
| nginx.ingress.kubernets.io/app-root       | str     | 定义Controller必须重定向的应用程序根，如果它在'/'上下文中    |
| nginx.ingress.kuberntes.io/use-regex      | boolean | 指定ingress上定义的路径是否使用正则表达式                    |
