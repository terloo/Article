# 探针
探针(Probe)是由kubectl对容器(包括InitC)执行的定期探测。

## 程序
1. 处理程序类型：
   - Exec：在容器内执行执行命令。如果命令退出时返回码为0则认为探测成功。
   - TcpSocket：在指定端口上能建立tcp连接，则认为探测成功
   - HttpGet：对指定的端口和路径上的容器的IP地址执行HTTP Get请求。如果有响应状态码200-399，则探测认为成功。
2. 探测结果：
   - 成功：容器通过了探测
   - 失败：容器未通过探测
   - 未知：探测失败，将不会采取任何措施。
3. 探测方案
   - StartupProbe：指示容器中的进程是否已经启动。startupProbe在探测时，会阻止LivenessProbe和ReadinessProbe的探测。startupProbe探测就绪之后，不再进行探测，同时开始LivenessProbe和ReadinessProbe的探测。
   - LivenessProbe：指示容器是否正在运行。如果存活探测失败，则controller会杀死容器，并且容器将执行重启策略。如果容器不提供存活探针，则默认为Success。存活探测会在就绪探测success并且初始等待时间之后开始进行。
   - ReadinessProbe：指示容器是否准备好服务请求。如果就绪探测失败，端点控制器将从与Pod匹配的所有Service的端点中删除Pod的IP地址，此时Pod的运行状态为Running，但并不被认为是Ready。初始延迟(initialDelaySecond)之前的**就绪状态**默认为Failure。如果容器不提供就绪探测指针，则默认为Success。

## 配置清单字段
三种探针字段相同，此处拿readiness举例。
| 字段名                                               | 字段类型 | 说明                                                      |
| ---------------------------------------------------- | -------- | --------------------------------------------------------- |
| spec.containers[].readinessProbe                     | Obj      | 探测方案                                                  |
| spec.containers[].readinessProbe.initialDelaySeconds | int      | Pod启动后延迟多少秒进行探测，默认0                        |
| spec.containers[].readinessProbe.periodSeconds       | int      | 探测间隔，默认10                                          |
| spec.containers[].readinessProbe.timeoutSeconds      | int      | 探测超时世界，默认1                                       |
| spec.containers[].readinessProbe.successThreshold    | int      | 探测失败后再次成功需要的阈值，默认1                       |
| spec.containers[].readinessProbe.failureThreshold    | int      | 探测失败的重试次数，默认3。最大重试次数后直接进行重启策略 |
| spec.containers[].readinessProbe.httpGet             | Obj      | 探测类型：httpGetAction                                   |
| spec.containers[].readinessProbe.httpGet.port        | int      | 端口                                                      |
| spec.containers[].readinessProbe.httpGet.path        | str      | 探测路由地址                                              |
| spec.containers[].readinessProbe.exec                | Obj      | 探测类型：execAction                                      |
| spec.containers[].readinessProbe.exec.command[]      | list     | 执行的命令                                                |
| spec.containers[].readinessProbe.tcpSocket           | Obj      | 探测类型：tcpSocketAction                                 |
| spec.containers[].readinessProbe.tcpSocket.port      | int      | 端口                                                      |

## 探针在Pod资源清单中的配置-httpGet就绪探测
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-httpget-pod
  namespace: default
spec:
  containers: 
  - name: contianer-nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet: 
        port: 80
        path: /iiiiidenx.html
      initialDelaySeconds: 1
      periodSeconds: 3
```

## 探针在Pod资源清单中的配置-Exec存活探测
```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: liveness-exec-pod
  namespace: default
spec:
  containers:
  - name: container-nginx
    image: busybox
    command: ['/bin/bash', 'touch /tmp/live; sleep 60; rm /tmp/live; sleep 60']
    livenessProbe:
      exec:
        command: ['test', '-e', '/tmp/live']
      initialDelaySeconds: 3
      periodSeconds: 3
```
