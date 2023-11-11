# context
context是go中用于管理goroutine上下文的一个标准库，可以实现上下文信息保存，超时管理，级联取消等功能

## 接口
```go
type Context interface {
    // 返回一个超时时间，如果ok为false代表无超时时间
	Deadline() (deadline time.Time, ok bool)

    // 返回一个chan，当从该channel获取到一个信号时，代表该context已结束
	Done() <-chan struct{}

    // 返回错误
	Err() error

    // 携带的上下文信息
	Value(key any) any
}
```

## emptyContext
Context接口的基础实现，没有任何功能，用做根Context
```go
// 不能使用struct{}，因为该类型的变量必须有不一样的地址
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key any) any {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// 根Context
func Background() Context {
	return background
}

// 占位使用的Context
func TODO() Context {
	return todo
}
```

## valueContext
valueContext用于保存上下文信息，只实现了Value方法，其余方法由父Context实现。由于valueContext的设计，**kv一旦被赋值便无法更改，保证了在数据争用场景下的安全性**。  
Value的值是隐含性质的，不能放需要显示使用的值。需要更新值时应该调用WithValue函数新建valueContext
```go
// 从父Context新建一个子valueContext的函数
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context
	key, val any
}

func stringify(v any) string {
	switch s := v.(type) {
	case stringer:
		return s.String()
	case string:
		return s
	}
	return "<not Stringer>"
}

func (c *valueCtx) String() string {
	return contextName(c.Context) + ".WithValue(type " +
		reflectlite.TypeOf(c.key).String() +
		", val " + stringify(c.val) + ")"
}

// 实现了方法，如果key不匹配，则递归查找自己的父Context
func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}

func value(c Context, key any) any {
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case *timerCtx:
			if key == &cancelCtxKey {
				return &ctx.cancelCtx
			}
			c = ctx.Context
		case *emptyCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```

## cancelContext
cancelContext实现了级联取消功能，主要用于IO密集型任务。
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}

func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}

func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}
```

## 使用cancelContext进行级联取消操作
```go
func main() {
	ctx, cancelFunc := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancelFunc()
	go A(ctx)

	<-ctx.Done()
	time.Sleep(time.Second)
}

func A(ctx context.Context) {
	go B(ctx)
	go B(ctx)
}

func B(ctx context.Context) {
	go C(ctx)
	go C(ctx)
}

func C(ctx context.Context) {
    // 传播context到request请求中
	request, _ := http.NewRequestWithContext(ctx, http.MethodGet, "https://www.google.com", nil)
	_, err := (&http.Client{}).Do(request)
    // request将在context到期后主动结束，释放该goroutine
	if err != nil {
		fmt.Println("request error", err.Error())
	}
}
```