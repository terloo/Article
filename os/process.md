# process
process(进程)是操作系统中资源分配、保护和调度的基本单位

## 进程内存空间组成
1. data：存放全局变量和静态变量
2. text：存放程序指令
3. stack：栈空间
4. heap：堆空间

## 进程状态
进程在执行期间自身状态会发生变化，进程具有三种状态
1. 运行态(Runtime)：进程的代码正在CPU上运行
2. 就绪态(Ready)：进程具备运行条件，等待分配CPU
3. 等待态(Waiting)：进程正在等待某些事件的发生(比如IO操作结束或者是一个信号)
4. 新建态(New)：进程刚被创建还未开始执行
5. 终止(Terminated)：进程已执行完毕

## 进程何时失去CPU
1. 内部事件
   1. 进程主动放弃(yield)CPU，进入等待/终止状态，如开始使用I/O设备，(非)正常结束等
2. 外部事件
   1. 进程被剥夺CPU使用权，进入就绪状态，这个动作称为抢占(preempt)，如时间片到达，高优先级进程到达等

## 进程切换
并发进程中，一个进程在执行过程中可能会被另一个进程替换占用CPU，这个过程称为进程切换。进程切换也称为进程上下文切换

### 切换时机
1. 进程需要进入等待状态
2. 进程被抢占CPU而进入就绪状态

### 切换过程
1. 保存被中断进程的上下文信息(Context)
2. 修改被中断进程的控制信息(如状态等)
3. 将被中断的进程加入相应的状态队列
4. 调度一个新的进程并恢复它的上下文信息

### 中断Interrupt
中断指程序执行过程中，当发生某个事件时，中止CPU上现行程序的运行，引出该事件的处理程序执行，执行完毕后返回原程序中断点继续执行。中断是用户态向核心态切换的唯一途径，系统调用实质上也是一种中断

### 中断源
1. 外中断：来自处理器之外的硬件中断信号，一般称为中断
   1. 如时钟中断、键盘中断、外围设备中断
   2. 外部中断均是异步中断
2. 内中断(异常Exception)：来自于处理器内部，指令执行过程中发生的中断，一般称为异常
   1. 硬件异常：掉电、奇偶校验错误等
   2. 程序异常：非法操作、地址越界、断点、除数为0
   3. 系统调用
   4. 内中断均是同步中断

## 特权指令和非特权指令
1. 特权指令Privileged Instructions，特权指令只能在内核模式下进行运行
   1. IO指令和停止(Halt)指令
   2. 关闭所有所有中断
   3. 设置定时器Timer
   4. 进程切换
2. 非特权指令Non-Privileged Instructions，非特权指令只能在用户模式下进行运行

## 进程控制块
Process Control Block(进程控制块，PCB)包含了许多进程相关的控制信息，每个进程都有一一对应的进程控制块
1. 进程状态
2. 进程编号PID
3. Program Counter：PC值，进程下一条要执行的指令
4. registers：进程所使用的寄存器的值
5. memory limits：内存地址限制
6. list of open files：进程打开的所有文件

## 进程队列
进程队列由一个就绪队列和多个属于不同模块的等待队列组成

## 进程调度
进程在整个生命周期中会在各个队列中迁移，由操作系统的一个调度器(scheduler)来执行

## 进程上下文
支撑一个进程运行的所有数据被称为进程的上下文，包括进程的PCB，栈、堆、text、data