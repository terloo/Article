# playbook

## 核心元素
1. Hosts：执行任务的远程主机列表
2. Tasks：任务集
3. Variables：内置变量或者自定义变量
4. Templates：模板，可替换模板文件中的变量并实现一些简单的逻辑操作
5. Handlers，notify：由特定条件触发的操作，满足条件才可执行，否则不执行
6. Tags：标签，指定某条任务执行，用于选择运行playbook中的某些代码
