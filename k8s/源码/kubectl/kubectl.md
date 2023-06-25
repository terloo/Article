# Kubectl

## 创建一个资源的流程
1. 创建cobra客户端所有命令子命令和子命令
2. 实例化一个Factory
3. 调用Factory的newBuilder()来生成Builder，用Builder将命令行参数转化为资源对象
4. 将资源对象通过HTTP发送给api-server
5. 通过多层匿名Visitor返回的errors来判断是否与api-server成功交互
