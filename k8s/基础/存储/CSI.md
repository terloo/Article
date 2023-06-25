# CSI
CSI是k8s制定的k8s存储协议

## 过程
1. create：由csi插件将PV创建
2. attach：由csi插件将PV挂载到node的文件系统中
3. mount：由kubelet将attch路径挂载到容器中