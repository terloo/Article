# crictl
crictl用于控制实现了cri的进程

## 端点
设置crictl连接的cri端点
1. 设置参数--runtime-endpoint和--image-endpoint
2. 设置环境变量CONTAINER_RUNTIME_ENDPOINT和IMAGE_SERVICE_ENDPOINT
3. 在配置文件--config=/etc/crictl.yaml中设置端点
```yaml
# /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: true
```