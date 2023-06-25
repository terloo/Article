# Pod调试

## 服务调试

### 本地应用调用集群中的服务
使用kubectl提供的端口转发功能，将集群中的Service代理到本地
`kubectl port-forward [svc|pod] -n <ns>`

### 代理本地应用到集群
使用开源软件Telepresence

## 容器环境调试
进入容器环境，命令一般使用/bin/bash来开启一个shell
`kubectl exec -it -c <container> <pod> -- <command>`

## 使用调式容器调试
使用Kubectl插件kubectl-debug