# 公网添加Node

1. 修改主节点的广播ip地址为公网地址，`kube-apiserver --advertise-address=<公网ip>`
2. 使cni能正常工作
   1. 修改主/从节点的kubelet节点ip为公网地址，`kubelet --node-ip=<公网ip>`
3. 执行`kubeadm token create --print-join-command`，获取join命令