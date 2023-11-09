# error
error是go内置的一个接口

## 接口
error的约定十分简单，只需要返回一个string来标识该error时
```go
type error interface {
    Error() string
}
```

## errors
errors是go中提供的用于error操作的标准库
```go
package errors

// 新建一个error，此处返回指针是为了调用方使用==进行比较时，避免字符串产生的碰撞照成的意外相等
func New(text string) error {
    return &errorString{text}
}

type errorString struct {
    s string
}

type (e *errorString) Error() {
    return e.s
}
```

## Sentinel Error
Sentinel Error即在包级别预定义的一些error，用户在使用此包提供的函数时，可以利用Sentinel Error使用`==`判断错误的具体类型
```go
func XX() {
	_, err := os.Stat("filepath")
	if err == os.ErrNotExist {
		
	}
}
```
Sentinel的缺点，**尽可能不要在非标准库中使用**
1. error中无法携带更多的上下文
2. Sentinel Error会成为API的公共部分
3. Sentinel Error在两个包之间建立了强依赖关系

## Error Type
定义一个结构体，实现error接口，该结构体称为Error Type。使用Error Type，可以在error中包含更多的上下文，调用者需要用switch来对error进行类型转化获取上下文
```go
type ErrorTypeA struct {
	context1 string
	context2 int
}

func (e *ErrorTypeA) Error() string {
	return fmt.Sprintf("Error: context:%s, context2:%d", e.context1, e.context2)
}
```
ErrorType的缺点
1. 会让包中定义的error称为公共类型，与调用者产生强耦合
2. 需要类型转化才能获取error中的上下文

## Opaque errors
Opaque errors，不透明错误。调用方无法得知具体的错误或者错误的类型，只能通过`!= nil`对错误进行判断。如果需要判断error的类型或者error中的上下文，调用方需要使用包中提供的函数进行处理
```go
// 不对外暴露类型
type errorTypeA struct {
	context1 string
	context2 int
}

func (e *errorTypeA) Error() string {
	return fmt.Sprintf("Error: context:%s, context2:%d", e.context1, e.context2)
}

// 对外暴露方法，也可以断言为错误实现的接口
func IsErrorTypeA(err error) bool {
	_, ok := err.(*errorTypeA)
	return ok
}
```

## 正确处理错误
1. 无错误正常代码，使之称为直线，而不是缩进代码
2. 尽量使用库中提供的方法来处理错误，而不是手动处理
3. 重复进行同一操作时，对error进行集中处理
4. 将错误与操作绑定到一个结构体，每个操作之间判断是否存在error，如果存在，则不进行操作

## wrap errors
在go语言的设计中，一个error只应当被处理一次，一但这个错误被处理，就不应该再继续返回给上层调用者。这样做会损失该errors的调用路径，所以可以使用`github.com/pkg/errors`提供的Wrap方法将error信息和调用栈层层封装和额外信息直到被处理
```go
func main() {
	config, err := ReadConfig()
	if err != nil {
		// 打印错误类型和原始错误(进行判定)
		fmt.Printf("original error: %T %v\n", errors.Cause(err), errors.Cause(err))
		// 使用%+v的形式，可以直接将栈信息打印出来
		fmt.Printf("stack trace:\n %+v\n", err)
		os.Exit(0)
	}

	fmt.Println(config)
}

func ReadConfig() (string, error) {
	config, err := ReadFile("/root/config")
	if err != nil {
		return "", errors.WithMessage(err, "read config fail")
	}
	return string(config), nil
}

func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		// 此处的错误是原始错误，通过Wrap方法可以保存其调用栈
		return nil, errors.Wrap(err, "open file fail")
	}

	c, err := io.ReadAll(f)
	if err != nil {
		// 此处的错误是原始错误，通过Wrap方法可以保存其调用栈
		return nil, errors.Wrap(err, "read file fail")
	}

	return c, nil
}

```
常用方法
1. New或Errorf：创建一个新的error，包含错误信息，应在业务代码出错误时进行使用
2. 直接返回或WithMessage或WithMessagef：当错误来自本项目的包中时
3. Wrap或Wrapf或WithStack：使用标准库或者第三方库发生错误时
4. Cause或者%+v格式输出：在需要获取原始错误和调用栈时进行使用
5. 在程序的顶部或者是工作goroutine顶部，使用`%+v`进行输出
6. Sentinel Error需要与Cause获取的错误进行判断

## 等值判定以外的错误判断
使用了wrap error的错误，由于错误已经成为了错误链，故无法直接使用等值判断进行判断。go1.13提供了若干方法，方便进行错误类型判断
```go
func () {
	// 通过Is来判断错误链中是否含有某错误，Is会递归调用错误的Unwarp方法尝试获取下层错误
	if errors.Is(err, ErrorNotFound) {

	}

	// 通过As来判断错误链中是否包含某错误类型
	if errors.As(err, &ErrorTypeA{}) {

	}

	// 通过%w来封装一个错误，而不是直接将错误解析为字符串
	err = fmt.Errorf("some error: %w", err)
}
```
1. Unwarp：自定义错误实现该方法后，能使用此方法返回下层错误
2. Is：判断错链中是否包含某错误，错误也可以自己实现Is接口，Is函数会调用实现了Is接口的Is方法
3. As：判断错链中是否包含某类型的错误，As函数会调用实现了As接口的As方法
4. `%w`：通过%w谓词，fmt.Errorf方法可以返回一个经过包装的错误