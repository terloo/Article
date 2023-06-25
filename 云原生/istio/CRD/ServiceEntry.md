# ServiceEntry
istio内部会维护一个服务注册表，可以使用ServiceEntry向其中加入额外的条目。通常这个对象用来启用对Istio服务网格之外的服务发出请求

ServiceEntry中使用hosts字段来指定目标，字段值可以是一个完全限定名，也可以是一个通配符域名。其中包含的白名单，包含一个或多个允许网格中服务访问的服务

只要ServiceEntry涉及到了匹配host的服务，就可以和VirtualService以及DestinationRule配合工作

## 示例
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: baidu
spec:
  hosts:
  - *.baidu.com
  ports:
  - name: http
    number: 80
    protocol: HTTP
  - name: https
    number: 443
    protocol: HTTPS
```