# SchemeBuilder
SchemeBuilder是多个参数为Scheme的函数的切片，在调用AddToScheme时，将会将所有已注册的函数循环执行一次

## 结构体
```go
// vendor/k8s.io/apimachinery/pkg/runtime/scheme_builder.go
// SchemeBuilder是对Scheme进行操作的函数的切片
type SchemeBuilder []func(*Scheme) error
```

## 方法
```go
// vendor/k8s.io/apimachinery/pkg/runtime/scheme_builder.go
// 构造SchemeBuilder的时候即可以传入对Scheme进行操作的函数
func NewSchemeBuilder(funcs ...func(*Scheme) error) SchemeBuilder {
	var sb SchemeBuilder
	sb.Register(funcs...)
	return sb
}

// 向SchemeBuilder中注册对Scheme进行操作的函数
func (sb *SchemeBuilder) Register(funcs ...func(*Scheme) error) {
	for _, f := range funcs {
		*sb = append(*sb, f)
	}
}

// 执行所有函数，如果有错误，立即返回
func (sb *SchemeBuilder) AddToScheme(s *Scheme) error {
	for _, f := range *sb {
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}
```

## 外部包对SchemeBuilder的使用
每个资源的的外部资源库中，每个版本均实例化了一个SchemeBuilder，将该版本下所有资源和meta公共资源注册到Scheme的方法，**没有注册default函数和conversion函数**  
```go
// vendor/k8s.io/api/apps/v1/register.go
var (
    // 新建一个SchemeBuilder，添加addKnownTypes
	SchemeBuilder      = runtime.NewSchemeBuilder(addKnownTypes)
	localSchemeBuilder = &SchemeBuilder
    // 调用该方法，传入Scheme即可将该版本所有资源及meta资源注册到传入的Scheme中
	AddToScheme        = localSchemeBuilder.AddToScheme
)

// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
    // 注册当前版本所有资源
	scheme.AddKnownTypes(SchemeGroupVersion,
		&Deployment{},
		&DeploymentList{},
		&StatefulSet{},
		&StatefulSetList{},
		&DaemonSet{},
		&DaemonSetList{},
		&ReplicaSet{},
		&ReplicaSetList{},
		&ControllerRevision{},
		&ControllerRevisionList{},
	)
    // 注册meta中公共资源，主要是各种Options
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```

## 内部包外部版本对SchemeBuilder的使用
**直接使用外部包外部版本的SchemeBuilder**，并将转化方法和默认方法注册进外部版本的SchemeBuilder中
```go
// pkg/apis/apps/v1/register.go
var (
    // 直接使用外部包外部版本的SchemeBuilder
	localSchemeBuilder = &appsv1.SchemeBuilder
	AddToScheme        = localSchemeBuilder.AddToScheme
)

// 注册Default函数
func init() {
	localSchemeBuilder.Register(addDefaultingFuncs)
}

// pkg/apis/apps/v1/zz_generated.conversion.go
// 注册conversion函数
func init() {
	localSchemeBuilder.Register(RegisterConversions)
}
```

## 内部包内部版本对SchemeBuilder的使用
新实例化一个SchemeBuilder，将内部版本资源对象注册进SchemeBuilder中
```go
// pkg/apis/apps/register.go
var (
    // 构造一个SchemeBuilder
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme = SchemeBuilder.AddToScheme
)

func addKnownTypes(scheme *runtime.Scheme) error {
    // 注册所有内部版本
	scheme.AddKnownTypes(SchemeGroupVersion,
		&DaemonSet{},
		&DaemonSetList{},
		&Deployment{},
		&DeploymentList{},
		&DeploymentRollback{},
		&autoscaling.Scale{},
		&StatefulSet{},
		&StatefulSetList{},
		&ControllerRevision{},
		&ControllerRevisionList{},
		&ReplicaSet{},
		&ReplicaSetList{},
	)
	return nil
}
```