# CoreDNS

## 概念
CoreDNS包含一个内存态DNS，以及一个controller，controller监听Service以及Endpoints的变化并配置DNS，客户端Pod在进行域名解析时，从CoreDNS查询服务对应的地址记录

## FQDN
全限定域名(fully qualified domain name)是k8s用于表示一个Pod在集群中的唯一域名，格式：  
`$service-name.$namespace-name.svc.$clusterdomain`
1. service-name|pod-name：Service对象名或者Pod对象名
2. namespace：命名空间
3. svc：固定字串
4. clusterdomain：可配置的集群范围的域名后缀。默认为`cluster.local`

## 实现原理
CoreDNS会在每个Pod中配置/etc/resolv.conf如下
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
1. nameserver：代表该Pod使用的DNS服务器，即CoreDNS的ip地址
2. options ndots:5：代表在进行域名解析时，如果解析失败并且域名的点字符小于5，将会自动进行域名补全操作
3. search：进行域名补全时，需要补全的字符串

## 不同服务类型的DNS记录
1. 普通Service：ClusterIP、NodePort、LoadBalancer类型的Service都拥有ClusterIP，CoreDNS会为这些Service创建FQDN格式的DNS的记录以及PTR记录，并为端口创建SRV记录
2. HeadlessService：HeadlessService没有CluterIP，CoreDNS会为此Service创建多条A记录，并且目标为每个Pod的IP，格式为`$podName.$svcName.$namespaceName.svc.$clusterdomain`
3. ExternalService：此类Service用来引用一个已存在的域名，CoreDNS会为此Service创建一个CName记录指向目标域名

## 实践
k8s作为企业基础架构的一部分时，k8s服务也需要发布到企业DNS，需要定制企业DNS controller
- 对于k8s中的服务，在企业DNS中同样创建A/PTR/SRV记录，通常解析地址是LoadBalancer VIP
- 针对Headless Service，在PodIP可全局路由的情况下，按需创建DNS记录
- Headless Service的DNS记录，不宜过多，否则对企业DNS冲击过大
- 服务在集群内通过CoreDNS寻址，在集群内通过企业DNS寻址，服务在集群内外要有统一标识