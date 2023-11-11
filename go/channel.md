# channel
channel是go中提供的用于goroutine间通讯的安全消息队列

## Timing Out
```go
func main() {
    timeout := make(chan bool, 1)
    go func() {
        time.Sleep(time.Second)
        timeout <- true
    }()

    select {
        case  <-ch:
        // 业务channel
        case <-timeout:
        // 超时channel收到消息后，停止等待业务ch的返回数据
    }
}
```

## Moving on
```go
func main() {
    ch := make(chan bool)
    go func() {
        time.Sleep(time.Second)
        ch <- true
    }()

    select {
        case <-ch:
        // 业务channel
        default:
        // channel如果被阻塞，直接放弃发送数据
    }
}
```

## Pipeline
利用channel实现生成器的功能
```go
func gen() <-chan int {
    out := make(chan int)
    go func() {
        for i = 0; true; i++ {
            out <- i
        }
        close(out)
    }()
    return out
}
```