# Interface
Plugins的统一接口

## 接口
```go
// vendor/k8s.io/apiserver/pkg/admission/interfaces.go
type Operation string

const (
	Create  Operation = "CREATE"
	Update  Operation = "UPDATE"
	Delete  Operation = "DELETE"
	Connect Operation = "CONNECT"
)

type Interface interface {
	// Handles returns true if this admission controller can handle the given operation
	// where operation can be one of CREATE, UPDATE, DELETE, or CONNECT
	// 如果该Plugin可以处理该操作，则返回true
	Handles(operation Operation) bool
}

// 变更准入控制器接口，可以改变请求中的资源对象
type MutationInterface interface {
	Interface

	// Admit makes an admission decision based on the request attributes.
	// Context is used only for timeout/deadline/cancellation and tracing information.
	Admit(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}

// 验证准入控制器接口，可以验证请求中资源对象的合法性
type ValidationInterface interface {
	Interface

	// Validate makes an admission decision based on the request attributes.  It is NOT allowed to mutate
	// Context is used only for timeout/deadline/cancellation and tracing information.
	// 如果验证未通过，则返回err
	Validate(ctx context.Context, a Attributes, o ObjectInterfaces) (err error)
}
```

## Plugin验证接口
实现了该接口的Plugin表明该Plugin需要某些字段才能正常工作，调用该接口来验证字段是否正确添加
```go
// vendor/k8s.io/apiserver/pkg/admission/interfaces.go
type InitializationValidator interface {
	ValidateInitialization() error
}
```

## 读取Plugin的配置文件
从总配置文件中读取每个Plugin的单独的配置文件
```go
// vendor/k8s.io/apiserver/pkg/admission/interfaces.go
type ConfigProvider interface {
	ConfigFor(pluginName string) (io.Reader, error)
}
```