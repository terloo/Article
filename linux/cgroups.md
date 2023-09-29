# cgroups
cgroups是kernel提供的机制，该机制可以根据需要把一系列任务及其子任务整合(或分隔)到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架

## 主要作用
cgroups主要目的是为不同用户层面的资源管理提供一个统一的接口，从单个任务的资源控制到整个系统的虚拟化
1. 资源限制：cgroups可以对任务使用的资源总额进行限制
2. 优先级分配：通过分配CPU时间片数量和磁盘IO带宽，实际上等同于控制了任务运行的优先级
3. 资源统计：cgroups可以统计任务对系统资源的利用量，比如CPU时长、内存用量等
4. 任务控制：cgroups可以对任务进行挂起和恢复

## 相关概念
1. Task(任务)：在linux中，内核本身的调度和管理并不对线程或者进行区分，只是根据clone时传入的参数不同从概念上区分进程和线程。故这里使用Task来表示系统的一个进程或者线程
2. Cgroup(控制组)：cgroups中资源控制以cgroup为单位进行实现。Cgroup表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入一个cgroup，也可以从一个cgroup迁移到另一个cgroup
3. Subsystem(子系统)：cgroups中的子系统就是一个资源控制调度器。比如cpu子系统可以控制cpu的时间分配，内存子系统可以控制内存的用量分配
   1. blkio：对块设备的IO进行限制
   2. cpu：对cpu的时间片分配进行限制。与cpuacct挂载在同一目录
   3. cpuacct：生成任务使用cpu的资源报告
   4. cpuset：给cgroup中的任务分配独立的cpu(多处理器系统)和内存节点
   5. devices：允许或禁止任务对某设备的访问
   6. freezer：暂停/恢复cgroup中的任务
   7. hugetlb：限制使用的内存页数量
   8. memory：对cgroup中任务的可用内存进行限制，并自动生成内存资源报告
   9. net_cls：使用等级识别符(classid)标记网络数据包，让linux流量控制器(tc指令)可以识别来自特定cgroup任务的数据包，并进行限制
   10. net_prio：允许基于cgroup设置网络流量(network traffic)的优先级
   11. perf_event：允许使用perf工具监控cgroup
   12. pids：限制任务的数量
4. Hierarchy(层级)：一系列的cgroups以一个树状结构排列而成，每个层级通过绑定对应的子系统进行资源控制。层级中的cgroup节点可以包含零个或者多个子节点，子节点继承父节点挂载的子系统

## 文件系统接口
cgroups已文件系统的形式提供接口，文件系统已挂载到linux系统中
```sh
$ mount | grep cgroup

# 根挂载，cgroups中的文件均存在于内存中，在启用systemd的系统中，该挂载路径会被systemd挂载并设置为只读，用户无法添加子系统
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,mode=755)
# systemd系统对cgroup的支持
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
# 子系统根级挂载，在启用systemd的系统中，子系统目录由systemd挂载并进行管理
cgroup on /sys/fs/cgroup/misc type cgroup (rw,nosuid,nodev,noexec,relatime,misc)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
```

## 查看进程所属的cgroup
每一行被冒号分割为三列，分别代表
1. cgroup树的id，与`/proc/cgroups`文件中的id对应。id为1的树仅代表名称
2. cgroup树绑定的所有subsystem，多个subsystem用逗号隔开
3. 进程在cgroup树中的路径，即进程所属的cgroup，该路径相对于挂载点。如果没有代表在此子系统中无此任务
```sh
$ cat /proc/<pid>/cgroup

13:hugetlb:/
12:blkio:/system.slice/sshd.service
11:memory:/system.slice/sshd.service
10:cpuset:/
9:perf_event:/
8:pids:/system.slice/sshd.service
7:devices:/system.slice/sshd.service
6:rdma:/
5:net_cls,net_prio:/
4:freezer:/
3:cpu,cpuacct:/system.slice/sshd.service
2:misc:/
1:name=systemd:/system.slice/sshd.service
```
