# InitC
Pod能够具有多个容器，应用运行在容器中，但是它也可能有一个或多个先于应用容器启动的Init容器。如果Pod的Init容器失败，Kubernets会不停的重启InitC，除非Pod的重启策略配置为Never。

## Init容器与普通容器非常像，除了以下两点：
1. Init容器总是运行到成功完成为止
2. 每个Init容器都必须在下一个Init容器启动之前成功完成。

## 因为Init容器具有与应用容器分离的单独镜像，所以它们的启动相关代码具有以下优势：
1. 它们可以包含并运行使用工具，但是出于安全考虑，是不建议在InitC中包含这些工具的。
2. 它们可以包含使用工具和定制化代码来安装，但是不能出现在应用程序的镜像中。例如，创建镜像没必要FROM另一个镜像，只需要在安装过程中使用类似sed、awk、python或dig这样的工具。
3. 应用程序镜像可以分离出创建和部署的角色，而没必要联合它们构建一个单独的镜像。
4. InitC使用Linux Namespace，所以相对应应用程序容器来说具有不同的文件系统视图。因此，它们能够访问Secret权限，而应用程序容器则不能。
5. InitC必须在应用程序容器启动之前运行完成，而应用程序容器是并行运行的，所以InitC能够提供一种简单的阻塞或延迟应用容器的启动方法，直到满足了一组先决条件。

## 注意事项
1. 在Pod启动过程中，Init容器会按顺序在网络和数据卷初始化之后启动。每个容器必须在下一个容器启动之前成功推出。
2. 如果InitC由于运行时或失败退出，将导致容器启动失败，它会根据Pod的restartPolicy策略进行重启。
3. 在所有的Init容器没有成功启动之前，Pod将不会变为Ready状态（保持`Init:m/n`的状态）。Init容器的端口将不会在Service中进行聚集。正在初始化中的Pod处于Pending状态，但应该会将Initializing状态设置为True
4. 如果Pod重启，所有的InitC将会全部重新执行。
5. 修改运行中Pod的InitC的非Image字段等价于重启该Pod。
6. InitC具有与应用容器的所有字段。除了readinessProde，因为InitC容器无法定义完成（completion）和就绪（readiness）之外的其他状态。这会在验证过程中强制执行。
7. 在Pod中的每个app和Init容器的名称必须唯一。名称相同将导致验证过程出现错误。

## InitC在Pod清单中的配置
```yaml
apiVersion: v1
kind: Pod
metadata: 
   name: pod-nginx
   labels: 
      type: test
spec: 
   initContainers:  # initC的定义
   - name: init-c
      imgae: busybox
      command: ['sh', '-c', 'echo the pod-nginx is initC && sleep 60']
   contianers:
   - name: container-nginx
      image: nginx
      command: ['sh', '-c', 'echo the container is running']
```
