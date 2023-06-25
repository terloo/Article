# redis配置

1. 启动时使用配置文件`redis-server [configfilepath]`
2. 在server中查看某个配置项`config get [configname]`
3. 在server中修改某个配置项`config set [configname]`

| key                             | 说明                      | 默认值     | 备注                                  |
| ------------------------------- | ------------------------- | ---------- | ------------------------------------- |
| port                            | 启动端口                  | 6379       |
| bind                            | 绑定主机地址              | 无         |
| databases                       | 数据库数量                | 16         |
| daemonize                       | 是否已守护进程形式启动    | on         |
| logfile                         | 日志文件名                | 不记录日志 |
| loglevel                        | 日志级别                  | verbose    | debug、verbose、notice、warning       |
| dir                             | 设置当前服务文件保存路径  | ./         | 包括日志和持久化文件                  |
| maxclient                       | 最大客户端连接数          | 无限制     | 链接数到达上限后，redis会关闭新的链接 |
| timeout                         | 客户端最大闲置等待时间    | 300        | 单位秒，如需关闭该功能，设置为0       |
| requirepass <password>          | 身份验证密码              | 无         |
| slaveof <masterip> <masterport> | 连接master                | 无         |
| masterauth                      | slave连接master验证       | 无         |
| repl-timeout <second>           | slave响应超时时间         | 60         | 单位：秒                              |
| repl-ping-slave-priod           | master ping slave时间间隔 | 10         |
| repl-backlog-size <size>        | master复制命令缓冲区大小         | 1mb            | 主从同步时增量同步缓冲区大小                                                 |
| include                         | 继承配置项，文件路径             | 无             | 维护公共配置信息，方便管理配置文件                                           |
| dbfilename                      | RDB持久化文件名                  | dump.rdb       |
| rdbcompression                  | 是否在存储到本地数据库时进行压缩 | yes            | LZF压缩                                                                      |
| rdbchecksum                     | 是否进行RDB文件的校验            | yes            |
| stop-writes-on-bgsave-error     | 在遇到错误时是否放弃本次bgsave   | yes            |
| save second changes             | 自动进行bgsave操作               | 无             |
| appendonly                      | 是否开启AOF                      | no             |
| appendfsync                     | AOF策略                          | everysec       |
| appendfilename                  | AOF文件名                        | appendonle.aof |
| auto-aof-rewrite-min-size       | aof重写策略                      | 32M            | aof_current_size大于该值时进行重写                                           |
| auto-aof-rewrite-percentage     | aof重写策略                      |                | $\frac{aof\_current\_size - aof\_base\_size}{aof\_base\_size}\geq$该值时重写 |  |
| maxmemory                       | redis最大使用物理内存的比例      | 0，表示不限制  | 一般设置在50%以上                                                            |


```conf
port 6380
daemonize yes
logfile "6379.log"
dir /var/log/redis/6379
dbfilename dump-6379.rdb
rdbcompression yes
rdbchecksum yes
stop-writes-on-bgsave-error yes
save 60 10
appendonly yes
appendfsync everysec
```
