# namespace

linux内核提供了一下几种namespace来提供一个资源隔离和限制的空间
1. mount：隔离了容器的文件系统
2. uts：隔离了容器的hostname和domainname
3. pid：隔离了容器的pid资源，保证进程以1号pid启动
4. network：隔离了容器的网络
5. user：隔离了容器的用户，提供了容器内部uid和gid在宿主机上的映射
6. ipc：隔离了容器进程与其他进程的进程间通信
7. cgroup：隔离了容器的cgroup试图，能让容器的cgroup试图从/路径出发

## docker使用的ns
| ns类型            | 系统调用参数  | 内核版本 | 用途                                                    |
| ----------------- | ------------- | -------- | ------------------------------------------------------- |
| Mount Namespace   | CLONE NEWNS   | 2.4.19   | 隔离进程看到的挂载点视图                                |
| UTS Namespace     | CLONE NEWUTS  | 2.6.19   | 隔离nodename和domainname                                |
| IPC Namespace     | CLONE NEWIPC  | 2.6.19   | 隔离System V IPC和 POSIX Message Queues，负责进程间通信 |
| PID Namespace     | CLONE NEWPID  | 2.6.24   | 隔离进程ID                                              |
| Network Namespace | CLONE NEWNET  | 2.6.29   | 隔离网络设备，IP地址和端口等网络栈                      |
| User Namespace    | CLONE NEWUSER | 3.8      | 隔离用户组ID                                            |

## namespace相关操作

### `/proc/<pid>/ns`
查看该进程所有ns

### `lsns [options]`
用于查看操作系统中存在的ns
1. `-t <type>`：只查看指定类型的ns
2. `-p <pid>`：只查看某个进程的ns
3. `-l`：以表格形式输出
4. `-r`：以原始数据形式输出

### `nsenter [options] [program [argument]]`
进入某个ns执行命令
1. `-t <pid>`：使用指定进程的ns，通过其他选项来指定是否进入ns
2. `-n`：进入net
3. `-m`：进入mount
4. `-p`：进入pid
5. program：默认执行命令为`/bin/sh`