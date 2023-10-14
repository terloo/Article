# route-reflector
route-reflector(路由反射器)是BGP中用于减少BGPPeer的连接数量的组件，可以将若干对等体申明为route-reflector，其他对等体只需连接route-reflector即可

## 无中断由mesh模式切换为route-reflector模式
1. 将一个节点或多个节点申明为route-reflector，这个节点必须是无法调度pod的，否则再申明后其上的pod将无法连接到集群网络
   1. `kubectl annotate node my-node projectcalico.org/RouteReflectorClusterID=<ID，一般为ip地址>`
   2. 给节点label方便进行选择`kubectl label node my-node route-reflector=true`
2. 创建BGPPeer资源对象，让所有非rr节点与rr节点进行配对
```yaml
kind: BGPPeer
apiVersion: projectcalico.org/v3
metadata:
  name: some.name
spec:
  nodeSelector: all()
  peerSelector: route-reflector == 'true'
```
3. 等待非rr节点与rr节点配对完成
4. 关闭mesh模式，没有bgpconfiguration时需要先创建
   1. `calicoctl patch bgpconfiguration default -p '{"spec": {"nodeToNodeMeshEnabled": false}}'`
5. 使该rr节点可调度