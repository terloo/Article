# CNI

## 概念
CNI(container network interface)是k8s制定的一个容器网络规范，k8s可以调用实现了该规范的CNI插件，来配置容器的网络。实现不同Pod之间容器的扁平化网络。

## kubelet调用cni插件的流程
1. 读取`/etc/cni/net.d/xxx.conf`来获取cni程序信息
2. 根据信息执行`/opt/cni/bin/xxx`可运行程序来配置容器网络
