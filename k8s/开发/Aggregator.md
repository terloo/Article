# Aggregator
Aggregator是k8s提供的外部资源服务器扩展机制，apiServer在收到api请求后，将会查询对应的apiService资源对象，如果apiService资源对象中配置的服务器为外部服务器，apiServer将会代理请求到外部服务器

## 环境
1. apiServer配置：
   1. `--requestheader-client-ca-file=/etc/kubernetes/pki/ca.crt`：CA证书
   2. `--proxy-client-cert-file=/etc/kubernetes/pki/front-proxy.crt`：apiServer发起客户端请求时使用的证书
   3. `--proxy-client-key-file=/etc/kubernetes/pki/front-proxy.key`：客户端证书对应的key
2. apiService资源对象
```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    k8s-app: metrics-server
  name: v1beta1.metrics.k8s.io
spec:
  # 需要代理的资源组
  group: metrics.k8s.io
  # 需要代理的资源版本
  version: v1beta1
  # 代理服务器端点
  service:
    name: metrics-server
    namespace: kube-system
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  versionPriority: 100
```
3. apiServer如果无法访问ClusterIP(apiServer节点上未运行kube-proxy)
   1. `--enable-aggregator-routing=true`使apiServer直接解析ClusterIP对应的PodIP
   2. 让Pod使用主机网络