# postStart和preStop
定义了容器创建后和终止前的钩子操作

## postStart
在容器创建之后，立即执行一个操作。postStart和容器命令**不严格**保证顺序。postStart结束前，Pod不会被标记为Running

## preStop
在容器进入Terminating状态(Complete状态不会执行)后进行的**同步**操作，执行完preStop之后再向容器进程发出SIGTERM信号  
如果GracePeried结束后preStop还未执行完毕，则再进入2s的小GracePeried。preStop执行完毕或GracePeried到期后，才会继续执行Pod的删除流程。

## 资源清单字段
均位于containers[].lifecycle下，postStart为例
| 字段                                                 | 值   | 说明                    |
| ---------------------------------------------------- | ---- | ----------------------- |
| spec.containers[].lifecycle.postStart                | Obj  | 主容器启动前执行的操作  |
| spec.containers[].lifecycle.postStart.exec           | Obj  | 钩子操作：command       |
| spec.containers[].lifecycle.postStart.exec.command[] | list | 执行的具体命令          |
| spec.containers[].lifecycle.postStart.httpGet        | Obj  | 钩子操作：httpGetAction |
| spec.containers[].lifecycle.postStart.httpGet.port   | int  | 端口                    |
| spec.containers[].lifecycle.postStart.httpGet.path   | str  | 检测路由地址            |
