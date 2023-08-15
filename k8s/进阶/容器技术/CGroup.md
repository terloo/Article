# CGroup
CGroup是一种用文件系统来进行组织的linux进程资源控制配置信息，默认的映射目录为`/sys/fs/cgroup`

## 功能
1. 限制进程组可以使用的资源数量(Resource limit)：例如memory子系统可以为进程组设定一个memory使用上限，一旦进程组使用的内存达到限额再申请内存，即触发OOM错误
2. 进程组的优先级控制(Prioritization)：例如可以使用cpu子系统为某个进程分配特定的cpu share
3. 记录进程组的资源数量(Accounting)：比如可以使用cpuacct子系统记录某个进程组使用的cpu时间
4. 进程组隔离(Isolation)：比如使用ns子系统可以使不同的进程组使用不同的namespace，以达到隔离的目的。不同的进程组有各自的进程、网络、文件系统挂载空间
5. 进程组控制(Control)：比如使用freezer子系统可以将进程组挂起和恢复

## 概念
1. 子系统(subsystem)：即CGroup根目录下的子文件夹，一个子系统就是一个资源控制系器。比如cpu子系统就是控制cpu时间分配的一个控制器。子系统必须附加(attach)到一个层级上才能起作用，一个子系统附加到某个层级后，这个层级上所有的控制组都受到这个子系统的控制
2. 控制组(control group)：即CGroup子系统以及子系统下的文件夹，控制组一但被创建将会被自动写入一系列子系统相关的属性
3. 属性：控制组中一系列与子系统相关的配置文件
4. 任务(task)：在CGroups中，任务就是系统的一个进程，task(pid)可以被添加进控制组中的CGroup.procs配置文件中，以表示该进程受到控制组的控制
5. 层级(herarchy)：控制组可以组织成层级的形式(目录树)，既一颗控制组的树。控制组树上的子控制组继承父控制组的特定属性

## 子系统
即CGroup目录的根目录下的文件夹，称为CGroup子系统
1. blkio：限制进程磁盘读写速率
2. cpu：限制进程cpu使用
3. cpuacct：统计进程使用的cpu信息
4. cpuset：限制进程使用的cpu
5. memory：限制进程内存使用
6. devices：限制了进程能访问的设备
7. freezer：限制进程在停止过程中fork出新进程而逃逸到宿主机
8. pid

## CPU子系统
1. cpu.cfs_period_us：配置时间长度周期，单位为us。默认情况下为100000(0.1s)
2. cpu.cfs_quota_us：用来配置当前CGroup在cfs_period_us时间中最多能使用的CPU时间数，单位为us。默认为-1，代表不控制。**cfs_quota_us/cfs_period_us代表控制组能使用的cpu的核数**
3. cpu.shares：控制组在CPU资源不足时能获得CPU使用时间的相对值，默认为1024。由于cpu资源是可以压缩的，这个值会在cpu饱和时决定对控制组所使用的cpu进行压缩的比例

## CPU acct系统
1. cpuacct.usage：统计控制组的cpu使用情况
2. cpuacct.stat：统计控制组的cpu使用时间，单位纳秒

## memory子系统
1. memory.limit_in_bytes：设置控制组能使用的最多的内存。默认为-1，代表不限制。如果超过该值，触发OOM
2. memory.soft_limit_in_bytes：该设置并不会阻止控制组使用内存，只是在内存超过限制时，优先回收超额的内存
3. memory.usage_in_bytes：该控制组当前使用的内存
4. memory.max_usage_in_bytes：该控制组最大使用的内存

## 在k8s中的应用
1. limit.cpu：使用cpu.cfs_period_us和cpu.cfs_quota_us进行实现
2. request.cpu：使用cpu.shares来实现cpu相对使用时间的功能
3. limit.memory：使用memory.soft_limit_in_bytes进行实现