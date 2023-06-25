# CRIServices
除了用protobuf定义的底层传输接口以外，cri-api还定义了一些用于封装底层传输接口的Services接口。  
用于规范k8s在使用CRI接口时的行为规范，如兼容不同版本和服务日志打印。

## 分类
RuntimeService：调用ContainerRuntime相关功能的接口
ImageManagerService：管理镜像相关功能的接口
