# systemd整合cgroups
systemd将cgroup提供的文件形式接口进行封装，并使用其进行资源管理

## systemd依赖cgroups
cgroups的两个方面：层级结构和资源控制。systemd必须依赖层级结构，可选依赖资源控制

## systemd挂载的cgroups系统
在系统的开机阶段，systemd会把所有支持的subsystem子系统挂载到`/sys/fs/cgroup`下
```sh
# ll /sys/fs/cgroup
dr-xr-xr-x 5 root root  0 Aug  6 11:59 blkio
lrwxrwxrwx 1 root root 11 Aug  6 11:59 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Aug  6 11:59 cpuacct -> cpu,cpuacct
dr-xr-xr-x 7 root root  0 Aug  6 11:59 cpu,cpuacct
dr-xr-xr-x 3 root root  0 Aug  6 11:59 cpuset
dr-xr-xr-x 5 root root  0 Aug  6 11:59 devices
dr-xr-xr-x 3 root root  0 Aug  6 11:59 freezer
dr-xr-xr-x 3 root root  0 Aug  6 11:59 hugetlb
dr-xr-xr-x 7 root root  0 Aug  6 11:59 memory
dr-xr-xr-x 2 root root  0 Aug  6 11:59 misc
lrwxrwxrwx 1 root root 16 Aug  6 11:59 net_cls -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Aug  6 11:59 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Aug  6 11:59 net_prio -> net_cls,net_prio
dr-xr-xr-x 3 root root  0 Aug  6 11:59 perf_event
dr-xr-xr-x 5 root root  0 Aug  6 11:59 pids
dr-xr-xr-x 3 root root  0 Aug  6 11:59 rdma
dr-xr-xr-x 5 root root  0 Aug  6 11:59 systemd
```
除了systemd目录，其余目录均是子系统。systemd目录是systemd维护的自己使用的非subsystem的cgroups层级结构。实际上systemd目录是用来实现`层级结构`特性的

## cgroup的默认层级
通过将cgroup层级系统与systemd unit树进行绑定，systemd可以把资源管理的设置从进程级别移动至应用级别  
因此可以使用systemctl指令或者修改unit配置文件来管理unit相关的资源  
默认情况下，systemd会自动创建三个unit层级来为cgroup树提供统一的层级结构
1. service：一个或一组进程，由systemd根据unit配置文件启动。多个进程可以作为一个整体被启停
2. scope：一个或一组由外部创建的进程。进程通过fork()启动和终止，之后被systemd在运行时注册的进程，scope会将其封装。例如用户会话、容器和虚拟机
3. slice：保存层级关系的unit。slice可以嵌套slicen，scope或者service作为嵌套的终点。可以通过`systemd-cgls`命令查看

## systemd使用cgroups
service、scope和slice会被直接映射为cgroup树中的对象。如corn.service属于system.slice，会被直接映射为system.slice/corn.service。在对service进行资源限制时，可以直接通过配置systemd unit配置文件，来使用cgroup进行限制
