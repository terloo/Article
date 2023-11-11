# sync
sync是go中提供的用于解决并发状态下数据同步问题的标准库

## data race
同一个数据，在没有同步语意保护的情况下，同时被两个goroutine进行写操作，称为数据争用(data race)。被数据争用的数据，最终状态是不可预测的。使用`go build -race`能在编译时进行data race分析

## machine word size
matchine word size(机器字长)是计算机能对内存/缓存进行一次原子写入的最大长度，64位计算机的机器字长为8B。如果某结构体的内存布局小于等于机器字长，那么可以认为该结构体不存在数据争用的问题
```go
func main() {
	ta := &TA{id: 10, name: "ta"}
	tb := &TB{name: "tb"}

    // interface变量结构体组成为8B的类型指针加8B数据指针，在并发赋值的情况下，有可能导致类型指针与数据指针的数据类型不一致，导致panic
	var maker Maker = ta

	var fun1, fun2 func()

	fun1 = func() {
		maker = ta
		go fun2()
	}

	fun2 = func() {
		maker = tb
		go fun1()
	}

	go fun1()

	for {
		fmt.Println(maker.Hello())
	}

}

type Maker interface {
	Hello() string
}

type TA struct {
	id int
	name string
}

func (a *TA) Hello() string {
	return "hello" + a.name
}

type TB struct {
	name string
}

func (b *TB) Hello() string {
	return "helloB" + b.name
}

```

## atomic
atomic包是sync库的子包，利用操作系统提供的Copy-On-Write(写时复制)功能来实现内存数据原子更新，利用atomic可以保护单独数据的并发更新。多适用于读多写少的环境。
```go
func main() {
	var config atomic.Value

	config.Store(loadConfig())

	go func() {
		for {
			// 每隔10s进行一次更新
			time.Sleep(10 * time.Second)
			config.Store(loadConfig())
		}
	}()

	for i := 0; i < 10; i++ {
		go func() {
			for {
				// 保证并发读数据时不会读到更新了一半的脏数据
				v := config.Load()
				fmt.Println(v)
			}
		}()
	}
}
```

## Mutex
Mutext三种工作模式：
1. Barging：吞吐量优先，当锁被释放时，会唤醒第一个等待者，然后把锁给第一个等待者或者第一个请求锁的人
2. Handsoff：公平优先，当锁被释放时，会唤醒第一个等待者，然后锁等待第一个等待者准备好之后将锁给第一个等待者
3. Spinning：自旋，第一个等待者不挂起gouroutine，而是循环等待，在锁释放后，获得锁
GO1.8采用Barging和Spinning结合得模式实现Mutex。在1.9之后，又引入了饥饿状态，在一个gouroutine被标记为饥饿状态时，将直接适用Handsoff模式获取锁，此时Spinning模式也会被禁用

## errgroup
errgroup是google提供的库，用于解决多个任务并发执行(fork-join)并返回错误的场景

## Pool
sync.Pool用于保存和复用临时对象，以减少内存分配，降低GC压力，适用于Request-Driven场景
1. Pool内部由ring buffer进行实现
2. 使用Get()获取对象，使用Put()放置对象
3. 由于Pool中的对象随时可能被GC回收，所以不能用于需要进行资源释放的对象(文件、连接等)