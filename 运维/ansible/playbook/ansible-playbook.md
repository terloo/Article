# ansible-playbook

## 命令格式
`ansible-playbook [options] <filename.yaml1> <filename.yaml2> ...`

## options
| 选项          | 说明                                 |
| ------------- | ------------------------------------ |
| --check,-C    | 只检测可能出现的变化，不真正执行操作 |
| --list-hosts  | 列出运行任务的主机                   |
| --list-tags   | 列出tags                             |
| --list-tasks  | 列出tasks                            |
| --limit hosts | 只对指定主机列表中的主机执行task     |
| -v -vv -vvv   | 显示详细日志                         |