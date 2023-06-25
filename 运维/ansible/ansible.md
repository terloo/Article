# ansible
ansible是一个对多态服务器批量执行命令或操作的工具

## 命令格式
`ansible [options] [host-group] [-m [module]] [-a [module args]]`
1. options：ansible命令选项
2. host-group：主机组，pattern匹配
3. module：指定的模块
4. module args：模块参数

## 文档查询
`ansible-doc -s [module]`
1. s：查找指定模块的文档