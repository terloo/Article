# runtime

## M:N模型
GO会创建M个线程，之后创建的N个Goroutine都会依附于这些线程进行执行，N的个数会远远超过M的数量，称为M:N模型。  
线程同一时间只能执行一个Goroutine，当当前执行的Goroutine发生阻塞时，runtime会切换另一个就绪的Goroutine继续进行执行

## GMP
1. G：Groutine的缩写，每次`go func()()`都会创建一个goroutine，没有数量限制。使用结构体`runtime.g`，包含了当前Groutine的状态、堆栈、上下文
2. M：工作线程(OS Thread)也被称为Machine，使用结构体`runtime.m`。
   1. 所有M都具有线程栈，如果不对该线程栈提供内存，操作系统会给该线程栈提供内存(不同操作系统提供的线程栈大小不同)
   2. 如果指定了线程栈，则M.stack指向G.stack，M的PC寄存器指向G提供的函数地址，进行G执行
3. P：Processor，代表了M所需的上下文环境，也是处理用户程序的逻辑处理器。它负责衔接M和G的调度上下文，将等待执行的G和M进行对接
   1. 当P有任务时需要创建或唤醒一个M来执行它队列中的任务，当P/M绑定后，构成一个执行单元
   2. P的个数应小于`GOMAXPROCS`的个数
   3. mcache/stackalloc从M中迁移到P中，而G队列也被分为两类，一类是全局G队列，一类是每个P的本地队列

## GM调度模型
早期go runtime只有G与M的抽象，G存储于全局队列中
1. 单一全局互斥锁(Sched.Lock)和集中状态存储，导致所有关于goroutine的操作都需要进行锁操作
2. Goroutine传递问题：M与M之间经常会反复传递可运行的Goroutine，导致调度延迟增大
3. Pre-M持有内存缓存：每个M都持有mcache和stackalloc，然而M只有在持有G并运行时，才会使用到这部分内存，M处于syscal并不需要
4. 内存亲缘性较差：由于G进行全局调度，调度到同一个M的几率太小，数据局部性不好
5. 严重的线程阻塞/解锁：在系统调用的情况下，工作线程经常被阻塞和取消阻塞，这增加很多开销。比如M找不到G，此时M就会频繁进入阻塞/解锁状态来进行逻辑检查，以便及时发现可执行的G

## Work-stealing
当一个P执行完本地队列所有任务之后，并且全局队列为空时。将会挑选一个P，从它的队列中窃取一半的G。全局队列不为空，则会从全局中获取(当前个数/GOMAXPROCS)个G。  
为了保证公平性，从随机位置上的P开始，且便利顺序也具有随机性

## 全局队列
新建G时，P的本地队列并且达到256个时，会有半数的G被放到全局队列中。阻塞的系统调用如果找不到可用的P，也会被放置到全局队列中。  
为了避免全局队列饥饿，P每次获取G时，会有1/61的概率从全局队列获取G，而不是P的本地队列

## syscall
P执行G调用syscall时会解绑P，然后M和G进入阻塞，而P此时的状态为syscall状态，表明P的G正在syscall中，这时P不能被调度给别的M。  
如果短时间内M阻塞完毕，那么M会优先获取该P，有利于数据局部性  
syscall结束后M按照以下逻辑执行直到满足其中一个条件
1. 尝试获取同一个P，恢复执行G
2. 尝试获取idle list中空闲的P，恢复执行G
3. 找不到空闲P，把G放回到全局队列，M放回到idle list

## 系统监视器 sysmon
系统监视器，称为sysmon，是一个跟M绑定的特殊Goroutine，会定时扫描。在执行syscall时，如果某个P的G执行时间超过一个sysmon tick(10ms)时，就会把P设置为idle，放入idle list，重新调度给需要的M

## idle list
M和P都拥有各自的idel list

## Spining thread
在以下两种情况时，M会进行自旋操作直到跳出该情况
1. M不带P，找P进行挂载
2. M带P，找G进行执行
自旋的线程数不超过GOMAXPROCS。1类型的自旋在进行一段时间后进入阻塞状态。当有1类型的自旋时，2类型的自旋M将一直自旋不进行阻塞。

## Network Poller
G在发起网络调用时不会阻塞M，而是调用gopark函数，由netpoller对其进行多路复用的操作，将当前正在执行的G保存起来，然后切换到新的G堆栈执行。  
以下几个场景下，将会调用netpoll()函数，将处于就绪状态的G调度回来，推到global queue中：
1. sysmon
2. schedule()
3. GC：start the world

## go park
gopark函数会将G置于waiting状态，显示等待goready唤醒，在poller，锁，chan中均有使用

## OS thread limit
GO在频繁进行syscall时，是无法限制底层线程M的数量的，在编码时需要注意程序是否可能创建大量的Blocking Thread

## G life cycle
1. 绑定M0和G0，M0为程序的主线程，G0负责调度，即schedule()函数
2. 创建P，绑定M0和P0，首先会创建GOMAXPROCS个P，存储在sched的空间链表中(pidle)
3. M0和G0会创建一个指向runtime.main()的G，并放到P0的本地队列
4. runtime.main()会启动sysmon线程；启动GC线程；执行init函数；执行main.main函数