# DaemonSet

## 资源清单示例
```yaml
apiVerison: app/v1
kind: DaemonSet
metadata: 
name: daemonset-nginx
labels: 
    app: daemonSet
spec:
selector:
    matchLabels:
    name: daemon-example
template:
    metadata:
    labels:
        app: daemon-example
    spec:
    containers:
    - name: nginx-daemonset
        image: nginx:1.7.9
```