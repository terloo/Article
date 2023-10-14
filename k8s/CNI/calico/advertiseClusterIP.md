# advertiseClusterIP
通常情况下，k8s的clusterIP只能在集群中进行使用，calico能将clusterIP通过BGP通告到集群外，避免了对专用负载均衡器的需求

## sample
```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.96.0.0/16
  - cidr: fd00:1234::/112
```

```sh
calicoctl patch bgpconfig default --patch '{"spec": {"serviceClusterIPs": [{"cidr": "10.0.0.0/24"}]}}'
```