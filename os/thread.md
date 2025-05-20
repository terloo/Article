# thread

## 定义
1. 线程是CPU利用的基本单位，包含一个线程id、一个PC、一个寄存器集合和一个栈
2. 线程间共享其所属进程的code、data和其他系统资源(如打开的文件、信号)
3. 拥有多线程的进程被称为轻量级进程，只拥有单线程的进程称为重量级进程

## 线程内存空间组成
1. text：代码，共享
2. data：全局和静态变量，共享
3. head：堆，可共享可独享
4. registers：使用的CPU寄存器，独享
5. stack：栈，独享

## 多线程的优点
1. 响应性
2. 资源共享
3. 经济
4. 可伸缩性

## 多线程模型
1. 用户线程ULT(User Level Thread)：在user mode模式下运行的线程，它的管理无需内核支持
2. 内核线程KLT(Kernel Level Thread)：在kernel mode下运行，由操作系统支持与管理
3. M:1模型：所有的ULT对应着同一个KLT，已废弃
4. 1:1模型：每一个UTL都对应着一个KLT
5. M:M模型：

## 三大线程库
1. POSIX PThread：用户线程库和内线线程库，PThreads是POSIX标注定义的线程创建与同步API，不同的操作系统对该标注的实现不尽相同
2. Windows Threads：内线线程库
3. Java Thread：依据所依赖的操作系统而定