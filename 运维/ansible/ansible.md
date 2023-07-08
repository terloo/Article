# ansible
ansible是一个对多态服务器批量执行命令或操作的工具

## 命令格式
`ansible [options] <host-pattern> [-m [module]] [-a [module_args]]`
1. options：ansible命令选项
2. host-pattern：主机组，pattern匹配，all代表所有主机，`~`后可以使用正则表达式
3. module：指定的模块
4. module args：模块参数

## 命令执行流程
1. 加载配置文件，默认`/etc/ansible/ansible.cfg`
2. 加载对应的模块文件，如`ping`
3. 生成对应的临时py文件，通过ssh远程传输到目标服务器，默认目录`$HOME/.ansible/tmp/ansible-tmp-数字/xxx.py`
4. 授予临时py文件x权限并执行
5. 删除临时py文件

## options
| 选项                   | 说明                    | 备注                           |
| ---------------------- | ----------------------- | ------------------------------ |
| --version              | 查看版本                |                                |
| -m module              | 指定模块，默认为command |                                |
| -a args                | 模块参数                | 格式 `'options1=xxx options2=yyy'` |
| -v,-vv,-vvv            | 查看详细执行过程        |                                |
| --list-hosts,--list    | 显示指定分组的主机清单  |                                |
| -k,--ask-pass          | 通过密码验证连接ssh     | 需要所有主机清单的密码一致     |
| -C,--check             | 仅检查，不执行          |                                |
| -T,--timeout=TIMEOUT   | 执行命令的超时时间      | 默认为10，单位s                |
| -U,--user              | ssh的用户               |                                |
| -b,--become            | 手动指定sudu的用户      |                                |
| --become-user=USERNAME | sudu的用户名            |                                |
| -K,--ask-become-pass   | 通过密码来sudo用户      |                                |
