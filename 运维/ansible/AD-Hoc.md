# AD-Hoc
AD-Hoc是ansible适用与使用ansible模块执行单条命令的场景

## ansible模块
| 模块    | 解释                                  | 备注                    |
| ------- | ------------------------------------- | ----------------------- |
| ping    | 检查ansible与主机的连接               |                         |
| shell   | 批量执行shell命令(脚本)               |                         |
| command | ansible默认模块，适用于执行简单的命令 | 不支持特殊符号          |
| script  | 分发脚本并执行                        |                         |
| file    | 创建、删除文件和目录                  |                         |
| copy    | 复制(远程分发)文件                    | 从ansible节点分发到其他节点 |