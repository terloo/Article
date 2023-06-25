# vlog
vlog是用户自定义的Info级别日志等级，根据用户指定的`v`或`-vmodule`选项的值决定是否输出，可以看作是作为debug等级的补充。

## 使用方法
`klog.V(n).Enable`会在n**小于等于**`v`选项的值时返回true，即该日志应当被记录。`v`选项默认值为0
```go
if klog.V(2).Enable {
    klog.Info("log this")
}

// 或者
klog.V(2).Info("log this")
```

## vmodule