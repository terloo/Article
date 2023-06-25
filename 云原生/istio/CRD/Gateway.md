# Gateway
istio的网关只配置4-6层的负载平衡属性，例如要公开的port和tls等，而将L7层流量路由使用VirtualService绑定网关来实现。  
由此，istio可以像管理网格中其他任何数据平面一样来管理网关。  
对应xDS中的LDS

## 示例
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: web-gw
spec:
  selector: # 在哪一个gateway上进行监听，这里是代理入栈流量，所以在ingressgateway上进行监听
    istio: ingressgateway
  servers: # server，即xDS中的LDS定义
  - hosts: # 监听所有域名，端口为80
    - "*"
    port:
      name: http-simple
      number: 80
      protocol: HTTP
    tls:
      mode: SIMPLE # 单向认证
      credentialName: xxxx    # tls证书的Secret资源名
    tls:       # 或者使用ingress中mount的文件加载证书
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key

---