# kube-prometheus
kube-prometheus是在k8s中使用prometheus作为监控实现方案的一整套套件。包括了监控、展示、报警等功能。

## 套件
1. PrometheusOperator：Prometheus的Operator，用于创建和管理Prometheus实例
2. 高可用Prometheus：高可用的Prometheus实例
3. NodeExporter：用于收集宿主机系统指标
4. blackboxExporter：用于收集通信指标
5. PrometheusAdaptor：将Prometheus提供的查询接口聚合到apiserver中形成k8s metrics api
6. kube-state-metrics：收集k8s中资源的元数据(**功能被PrometheusAdaptor覆盖**)
7. 高可用AlertManager：高可用的AlertManager实例
8. Grafana：将Prometheus收集的数据进行图形化展示

## 部署
```
git clone https://github.com/prometheus-operator/kube-prometheus.git
kubectl kustomize . | kubectl create -f -
```