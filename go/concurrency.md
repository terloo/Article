# concurrency

## 管理一个gorountine的生命周期
管理一个gorountine的生命周期主要需要关注两点
1. 这个goroutine什么时候会结束，结束后如何让调用者感知到
2. 如何主动通知这个goroutine结束
```go
import (
	"fmt"
	"net/http"
	_ "net/http/pprof"
)

func serverApp() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {

		fmt.Fprintln(writer, "Hello world")
	})

	http.ListenAndServe(":8080", mux)
}

func serverDebug() {
	http.ListenAndServe("8081", http.DefaultServeMux)
}

func main() {
    // serverDebug如果异常退出，main函数无法感知到，会导致应用程序不符合预期
    // 如果使用log.Fatal会导致defer函数无法执行
	go serverDebug()
	serverApp()
}
```

## 防止goroutine泄露
如果一个goroutine一直处于运行或阻塞状态，且外部无法通知其结束，则称为该goroutine泄露
```go
func leak() {
    // leak函数一旦返回，则再无方法能向ch1中写入
	ch1 := make(chan struct{})

	go func() {
        // goroutine在此处阻塞，无法退出
		a := <-ch1
		fmt.Println(a)
	}()

    return
}
```

## 控制一个行为的超时时间
如果进行一个操作需要的时间是不可预估的，那么就需要对其最大时间进行控制
```go
func process(term string) error {
	ctx, cancelFunc := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancelFunc()

	done := make(chan struct{})

	go func() {
        // 将context传播给下层，下层也可以退出
		_ := dosomething(ctx, term)
		done <- struct{}{}
	}()

	select {
	case <- done:
		fmt.Println("dosomething completed")
		return nil
	case <- ctx.Done():
        // 超时时间结束后直接返回
		return errors.New("dosomething timeout")
	}
}
```

## 将并发行为交由调用者控制
被调用者不应该使用goroutine来执行自身的逻辑，而应该交由调用者来并发该逻辑