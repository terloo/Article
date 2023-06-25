# Requests和Limits
requests和limits是k8s中用于分配容器所需资源和使用资源限制的机制

## 资源类型
1. CPU：CPU的Request和Limit是通过CPU数来度量的。CPU的资源值是绝对值，而不是相对值。一般使用500m来代表0.5个CPU
2. Memory：内存的度量单位是字节数。使用整数或者定点整数加上国际单位制来标识内存值，大小写敏感，一般使用二进制单位
   1. 十进制单位：E、P、T、G、M、K、m(千分一)
   2. 二进制单位：Ei、Pi、Ti、Gi、Mi、Ki
3. 自定义资源：需要第三方插件对容器进行device挂载的资源，如GPU

## Pod资源限制总和
Pod资源定义总和 = Init容器资源限制最大值 + 主容器资源限制总和

## 主容器资源限制
1. 在只定义limits时，将会自动填充与limits相同的requests值
2. 在只定义requests时，limits值视为节点的资源最大值
```yaml
apiServer: v1
Kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"
        cpu: "1000m"
```

## Init容器资源限制
1. kube-scheduler调度带有多个init容器的Pod时，只计算requests最多的容器，而不是计算总和
2. kube-scheduler在计算该节点被占用的资源时，**即使init容器已经执行完毕，也会被纳入计算**。因为init容器在特殊情况下可能会被再次执行
```yaml
apiServer: v1
Kind: Pod
metadata:
  name: frontend
spec:
  initContainers:
  - name: init1
    image: busybox
    command: ["/bin/sh", "-c", "ls"]
    resources:
      requests:
        cpu: 10m
        memory: 10Mi
  - name: init2
    image: busybox
    command: ["/bin/sh", "-c", "ls"]
    resources:
      requests:
        cpu: 10m
        memory: 10Mi
  containers:
  - name: db
    image: mysql
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```