# 资源版本转化Converter
k8s使用内部版本当做中间版本的形式，来进行版本转化。

## Converter 结构体
```go
// vendor/k8s.io/apimachinery/pkg/conversion/converter.go
// Converter保存了k8s系统中所有的转化函数，被scheme所使用
type Converter struct {
	// Map from the conversion pair to a function which can
	// do the conversion.
	conversionFuncs          ConversionFuncs
	generatedConversionFuncs ConversionFuncs

	// Set of conversions that should be treated as a no-op
	ignoredUntypedConversions map[typePair]struct{}
}

type ConversionFuncs struct {
    // 映射关系，原类型到目标类型的结构体 -> 所使用的转化函数
	untyped map[typePair]ConversionFunc
}

// 原类型到目标类型的结构体
type typePair struct {
    source reflect.Type
    dest   reflect.Type
}

// 参数必须是指针，否则会返回error。Scope可以在调用转化函数时保存一些上下文信息
type ConversionFunc func(a, b interface{}, scope Scope) error

// 多次转化机制的定义(递归调用转化函数)
type Scope interface {
	// Call Convert to convert sub-objects. Note that if you call it with your own exact
	// parameters, you'll run out of stack space before anything useful happens.
	Convert(src, dest interface{}) error

	// Meta returns any information originally passed to Convert.
	Meta() *Meta
}
```
1. conversionFuncs：默认转化函数。这些转换函数一般定义在资源内部目录下的conversion.go中
2. generatedConversionFuncs：自动生成的转化函数。一般定义在资源内部目录下的zz_genrated.conversion.go
3. ignoredUntypedConversions：忽略在该字段注册的资源的转化操作

## 注册转化函数
Converter需要注册才能在k8s内部进行使用。目前k8s支持以下几种注册转化函数的方式
1. `scheme.AddIgnoredConversionType(from, to interface{})`：注册忽略的资源转化操作
2. `scheme.AddConversionFunc(a, b interface{}, fn conversion.ConversionFunc)`：注册默认的资源转化函数
3. `scheme.AddGeneratedConversionFunc(a, b interface{}, fn conversion.ConversionFunc)`：注册自动生成的资源转化函数
4. `scheme.AddFiledLabelConversionFunc(gvk schema.GroupVersionKind, conversionFunc FiledLableConversionFunc)`：注册字段标签的转化函数。FiledLableConversionFunc是将字段选择器转化为内部版本的函数