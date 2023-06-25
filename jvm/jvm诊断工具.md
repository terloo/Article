# JVM诊断工具

## jps(java process status)

查看正在运行的jvm进程，显示进程PID和主类名
| 选项名 | 说明                   | 备注                |
| ------ | ---------------------- | ------------------- |
| -q     | 只显示虚拟机id(PID)    |                     |
| -l     | 输出主类全名           | jar包的话显示全路径 |
| -m     | 输出main方法的参数     |                     |
| -v     | 输出虚拟机启动时的参数 |                     |
> 如果虚拟机使用-XX:-UsePerfData，则无法进行监控

## jstat(JVM Statistics Monitoring Tool)

用于监视JVM各种运行状态信息，可以显示进程中的类装载、内存、垃圾收集、JIT编译等运行数据  
基本语法: `jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]`
1. -t: 输出信息前加上时间戳，程序启动时为0，单位s
2. -h: 输出多少行数据后输出一次表头
3. vmid: jvm的id
4. interval: 输出时间间隔ms
5. count: 输出的总行数
6. option: 
   1. -class: 类的装载卸载信息和用时
   2. -compiler: 显示JIT编译器编译过的方法、耗时等信息
   3. -printcompilation: 输出已经被JIT编译的方法
   4. -gc: 显示与GC相关的堆信息。包括Eden、Survivor0和1、老年代，元空间的容量、已用空间、GC总时间等信息
   5. -gccapacity: 主要关注java堆各个区域使用到的最大、最小空间
   6. -gcutil: 主要关注空间的使用百分比
   7. -gccause: 与-gcutil类似，额外输出导致最后一次或者当前正进行的gc的原因
   8. -gcnew: 显示新生代状态
   9. -gcnewcapacity: 与-gcnew类似，主要关注使用到的最大、最小空间
   10. -gcold: 显示老年代gc情况
   11. -gcoldcapacity: 与-gcold类似，主要关注使用到的最大、最小空间
   12. -gcmetacapacity: 显示元空间使用到的最大、最小空间
-gc系类涉及到的表头:
1. S0C: survivor0区域的容量
2. S0U: survivor0已经使用的容量
3. EC: Eden区的总容量
4. EU: Eden区已经使用的容量
5. OC: 老年代总容量
6. OU: 老年代已使用容量
7. MC: 元空间的容量
8. MU: 元空间已经使用的容量
9. CCSC: 压缩类的总容量
10. CCSU: 压缩类使用的容量
11. YGC: Young gc的次数
12. YGCT: Young gc花费的时间
13. FGC: full gc的次数
14. FGCT: full gc花费的时间
15. GCT: gc花费的总时间
16. LGCC: 产生gc的原因


## jinfo(Configuration Info for java)
查看虚拟机的配置参数信息，也可以用于调整虚拟机的配置参数
基本语法：`jinfo <option> <pid>`
1. pid: 虚拟机id
2. option
   1. -sysprops: 可以查看System.getProperties()中的参数
   2. -flags: 查看曾经赋值过的一些参数
   3. -flag 具体参数名: 查看某个java进程的具体参数值
   4. -flag [+|-] 具体参数名: 修改某个boolean类型参数的值
   5. -flag 具体参数名=具体参数值: 修改某个非boolean类型参数的值 
> 只有`java -XX:+PrintFlagsFinal -version | grep manageable`中的值才可以动态修改


## jmap(JVM Memory Map)
导出内存映像文件和内存使用情况
基本语法：`jmap <option> <pid>`
1. pid: jvm的id
2. option: 
   1. -dump: 生成内存dump文件
   2. -heap: 显示堆空间相关信息，包括配置信息、容量使用情况
   3. -histo：输出堆中的对象统计信息，包括类、实例数量和合计容量。-histo:live只统计堆中存活的对象
   4. -clstats：类加载统计
   5. -finalizerinfo: 打印正在等待执行final方法的对象
> 导出内存文件  `jmap -dump[:live,]format=b,file=<filename> <pid>`
