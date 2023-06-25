# kube-state-metrics
kube-state-metrics用于收集k8s中资源的元信息的Exporter，由k8s项目提供

## 部署

### 单独部署
部署kue-state-metrics，以及相关的Service和RBAC。一般使用kube-prometheus进行部署，不进行单独部署
```
git clone https://github.com/kubernetes/kube-state-metrics.git
kubectl apply -f examples/standard
```

## 监控指标
```
ls docs
```