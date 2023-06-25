# PrometheusOperator
作为kube-prometheus监控框架的核心组件。prometheus operator负责监控相关的CRD的变化，并根据这些资源的变化来动态的管理prometheus的配置。  

## 部署
[prometheus-operator.yaml](prometheus-operator.yaml)

## 所使用的CRD
1. Prometheus：创建和管理Prometheus Server实例
2. ServiceMonitor：通过Service来进行Pod的服务发现
3. PodMonitor：匹配指定的Pod进行监控
4. PrometheusRule：管理告警配置
5. AlterManager：创建和管理AlterManager实例

## Prometheus
PrometheusOperator在检测到Prometheus的CRD时，将会根据此CRD来自动创建并配置Prometheus实例

### 创建Prometheus实例
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: inst
  namespace: monitoring
spec:
  resources:
    requests:
      memory: 400Mi
```

### Prometheus实例与ServiceMonitor配置项关联
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: inst
  namespace: monitoring
spec:
  serviceMonitorSelector:
    matchLabels: # 标签匹配ServiceMonitor
      monitoring: "true"
  resources:
    requests:
      memory: 400Mi
```

### 创建Prometheus实例绑定RBAC
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: inst
  namespace: monitoring
spec:
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels: # 标签匹配ServiceMonitor
      monitoring: "true"
  resources:
    requests:
      memory: 400Mi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

## ServiceMonitor
ServiceMonitor用于修改Prometheus中的监控配置项。默认情况下ServiceMonitor和监控对象在相同Namespace下。
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  namespace: monitoring
  labels:
    monitoring: "true"
spec:
  # 监控指定命名空间下指定标签的Service
  namespaceSelector:
    # any: "true"   匹配所有的命名空间
    matchNames:
    - default
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: web
    # 也可以使用Secret进行basicAuth认证
    # basicAuth:
    #   password:
    #     name: basic-auth
    #     key: password
    #   username:
    #     name: basic-auth
    #     key: user
```

## 示例Pod
```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: web
    port: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  selector:
    matchLabels:
      app: example-app
  replicas: 3
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: fabxc/instrumented_app
        ports:
        - name: web
          containerPort: 8080
```
