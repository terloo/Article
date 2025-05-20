# synchronization

## 并发进程之间的关系
1. 独立关系
2. 交互关系
   1. 并发进程执行过程中需要共享或者交换数据
   2. 交互的并发进程之间又存在着竞争(race)和协作(cooperation)的关系，竞争是协作关系的一个特例

## 竞争条件 Race Condition
多个进程并发操作同一个数据导致执行结果依赖于特定的进程执行顺序

## 同步机制
1. 进程同步指的是在进程协作中维护数据一致性的机制
2. 同步机制的工具
   1. Mutex lock 互斥锁，用于解决竞争
   2. Semaphore  信号量，用于解决协作

## 临界区 critical section
1. 进程中用于操作公共数据的代码片段，称为临界区
2. 为了保证并发安全，在同一时刻，不允许有两个以上的进程同时执行临界区代码
3. 为了解决临界区问题，提出了临界区协议

## 临界区协议
1. 进程在进入临界区前，需要在entry section请求许可
2. 进程在离开临界区前，需要在exit section归还许可

## 临界区管理准则
1. Mutual exclusion(Mutext)：互斥
2. Progress：前进
3. Bounded waiting：有限等待

## 软件层面解决临界区问题
1. 实现需要较高的编程技巧
2. 两个进程的实现代码是不对称的，当处理超过两个进程时，代码的复杂度会非常大
3. 两个著名的软件方案
   1. Peterson
   2. Dekker

## Mutex Lock
互斥锁是一个用于解决临界区问题的操作系统级别工具
1. 在进入临界区时获取这把锁，在一个进程获得锁时，其他进程无法获得这把锁
2. 在离开临界区时释放这把锁

## 原子操作 Atomic Operations
原子操作(又称原语)意味着这个操作在运行时是不可以被中断的，原子操作是操作系统的重要组成部分，下面两条硬件指令都是原子操作，可以用来进行临界区管理
1. test_and_set(bool* test)：将test的值修改为false，并返回原始test的值
```c
// c语言模拟硬件操作，该操作是原子的
bool test_and_set(bool* test) {
   bool result = *test;
   *test = false;
   return result;
}

// 借由test_and_set，使用c语言实现锁操作操作
bool available = true; // 初始值为unlocked状态
lock() {
   while(!test_and_set(&available)) {
      // do nothing
   }
}

unlock() {
   available = true;
}
```
2. compare_and_swap()：比较并交换

## Busy Waiting
1. 忙式等待是指占用CPU执行空循环实现等待
2. 这种类型的互斥锁也被称为自旋锁(spin lock)
   1. 缺点：浪费CPU周期
   2. 优点：进程在等待时没有上下文切换，对于等待时间不长的进程，自旋锁可以接受。在多处理器上优势更加明显

## 信号量 Semaphore
信号量是一种比互斥锁更强大的同步工具，它可以提供更高级的方法来同步并发进程  
信号量是一个整型变量，除了初始化操作()，只能进行两个标准的原子操作
1. P：wait操作
2. V：signal操作
```c
// c语言模拟硬件操作，PV操作均为原子操作
P(s) {
   while(s <= 0) {
      // do nothing
   }
   s--;
}

V(s) {
   s++;
}
```

## 二值信号量 binary semaphore
二值信号量的值只能是1或者0，通常将其初始化为1，用于实现互斥锁功能

## 一般信号量 counting semaphore
一般信号量的取值可以是任意数值，可以初始化为任意非负整数，用于控制并发进程对资源的访问

## Bounded-Buffer problem 有界缓冲区问题
又称为生产-消费模型，生产者消费者通过一个有限大小的缓冲区连接。在这个模型中，要使用信号量来协调生产消费者，并使用锁来保护buffer这个共享资源
```c
// 长度为k的缓存区
item B[k];
// 两个信号量，empty表示缓冲区是否可以放置元素
semaphore empty = k;
// full表示缓冲区是否可以消费元素
semaphore full = 0;
// 用于表示缓冲区的生产消费索引
int in = 0;
int out = 0;
// 互斥锁
semaphore mutex = 1;


Process producer_i() {
   P(empty);
   e = produce();
   P(mutex);
   B[in] = e;
   in = (in + 1) % 6;
   V(mutex);
   V(full);
}

Consumer consumer_i() {
   P(full);
   P(mutex)
   e = B[out];
   consume(e);
   out = (out + 1) % 6;
   V(mutex);
   V(empty);
}
```

## readers-writers   读者-写者问题
在某些资源竞争的场景中，某些进程(readers)只会对资源进行读操作，这些进程可以同时刻访问资源。  
而另一些进程(writers)会对进程进行写操作，这些进程无法与任何其他类型的资源共享资源
|     | R    | W    |
| --- | ---- | ---- |
| R   | 共享 | 互斥 |
| W   | 互斥 | 互斥 |

```c
semaphore rw = 1;
// 表示正在进行读文件的读者数量
int reader_count = 0;
// 用于保护reader_count资源
semaphore r_mutex = 1;

reader() {
   P(r_mutex);
   if (reader_count == 0) {
      reader_count++;
      P(rw);
   }
   V(r_mutex);

   P(r_mutex);
   read_file();
   reader_count--;
   if (reader_count) {
      V(rw);
   }
   V(r_mutex);
}

writer() {
   P(rw);
   write_file();
   V(rw)
}
```