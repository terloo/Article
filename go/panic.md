# panic
panic在go语言中，用于表示一个fatal级别的错误，此错误理论上无法进行处理。产生此错误后，程序原则上应立即退出防止出现更大的错误

## 处理野生goroutine
对于一个直接执行函数的goroutine，外层的recover()无法捕捉此goroutine抛出的panic，导致程序直接退出，需要进行处理
```go
func XX() {

    go func() {
        // 在每个野生goroutine中，defer一个执行了recover的函数，将panic处理为error
        defer func () {
            if err := revocer(); err != nil {

            }
        }

        panic("xxx")
    }()

}
```