# pprof
pprof是go中提供的用于go程序故障排查、性能调优的工具  
pprof读取Protocol Buffer v3的描述文件，它描述了一组callstack和symbolization信息，并生成报告和可视化网页，以帮助分析数据

## 性能分析
1. CPU：按照一定频率采集CPU当前使用情况，可以分析应用程序消耗CPU的主要位置
2. 内存：在应用程序进行堆分配时保存当前的栈信息，用于监视当前和历史内存使用情况，也可以排查内存泄漏
3. 阻塞分析：记录goroutine阻塞等待同步(包括channel和定时器)的位置
4. 锁分析：互斥锁分析，分析锁竞争情况
5. goroutine：分析goroutine运行情况

## 数据来源
能从三个方向进行pprof数据采集
1. 硬编码
2. Http Server
3. 基准测试

### 硬编码
通过在应用程序中引入`runtime/pprof`包，并硬编码埋点的方式来采集pprof数据，适合非server类型的应用程序或者一个有限代码块进行使用
```go
// Create a CPU profile file
f, err := os.Create("profile.prof")
if err != nil {
    panic(err)
}
defer f.Close()

// Start CPU profiling
if err := pprof.StartCPUProfile(f); err != nil {
    panic(err)
}
defer pprof.StopCPUProfile()
```

### Http Server
```go
import (
	"net/http"
	_ "net/http/pprof" // init函数会往默认的http mux中注入路由供获取对应的pprof
)

func main() {
    // 设置pprof采样参数
	runtime.SetMutexProfileFraction(1)
	runtime.SetBlockProfileRate(1)

	go func() {
		if err := http.ListenAndServe(":8080", nil); err != nil {
			log.Fatal(err)
		}
		os.Exit(0)
	}()

}
```

### 基准测试
通过go test工具，在进行基准测试过程中采样，生成pprof文件
```sh
go test -cpuprofile cpu.prof -memprofile mem.prof -mutexprofile mutex.prof -blockprofile block.prof -bench .
```

## 数据展示
pprof文件能经过工具处理后，能以一下三种形式展示以供分析
1. 报告
2. 交互式终端
3. Web页面

### 报告
通过命令`go tool pprof <format> <source>`可以生成性能报告，生成的报告可以有多种格式(至少选择一种)，格式可以是`text``pdf``gif``png``tree``web`等，部分格式需要Graphviz支持

### 交互式终端
通过命令`go tool pprof <source>`可以得到一个交互式中断，通过`top``list``web`命令可以查看数据

### web页面
通过命令`go tool pprof -http [host]:[port] <source>`可以启动一个web服务，通过web页面来查看数据
