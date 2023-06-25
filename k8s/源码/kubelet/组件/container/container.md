# contianer
kublet.container包是kubelet中用于处理节点中容器的所有组件

## 组件
1. runtime：CRI规范的包装类，kubelet用于与container runtime交互的顶层类
2. cache：缓存节点中所有容器的当前状态，以pod分类
3. container_gc：调用runtime进行废弃容器回收
4. sync_result：CRI同步操作的结果封装
5. os：go中os包的门面封装
6. helper：一些工具类的封装，和一个RuntimeHelper接口(由kubelet实现)