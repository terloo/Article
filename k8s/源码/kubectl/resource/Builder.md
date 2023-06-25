# Builder
Builder用于将命令行的参数转化为Visitor，通过Vistor可以将命令行参数转化为资源对象List

## 结构体
Builder结构体保存了命令行获取的各种参数，并通过不同的方法处理不同的参数，将其转为资源对象。
```go
// k8s.io/cli-runtime/pkg/resource/builder.go
type Builder struct {
	categoryExpanderFn CategoryExpanderFunc

	// mapper is set explicitly by resource builders
	mapper *mapper

	// clientConfigFn is a function to produce a client, *if* you need one
	clientConfigFn ClientConfigFunc

	restMapperFn RESTMapperFunc

	// objectTyper is statically determinant per-command invocation based on your internal or unstructured choice
	// it does not ever need to rely upon discovery.
	objectTyper runtime.ObjectTyper

	// codecFactory describes which codecs you want to use
	negotiatedSerializer runtime.NegotiatedSerializer

	// local indicates that we cannot make server calls
	local bool

	errs []error

	paths      []Visitor
	stream     bool
	stdinInUse bool
	dir        bool

	labelSelector     *string
	fieldSelector     *string
	selectAll         bool
	limitChunks       int64
	requestTransforms []RequestTransform

	resources []string

	namespace    string
	allNamespace bool
	names        []string

	resourceTuples []resourceTuple

	defaultNamespace bool
	requireNamespace bool

	flatten bool
	latest  bool

	requireObject bool

	singleResourceType bool
	continueOnError    bool

	singleItemImplied bool

	schema ContentValidator

	// fakeClientFn is used for testing
	fakeClientFn FakeClientFunc
}
```
1. categoryExpanderFn：获取一个categoryExpander的函数，categoryExpander可以将资源对象的类别名扩展为多个资源对象的资源名
2. mapper：用于反序列化资源对象，并构建一个Info对象
3. clientConfigFn：用于获取一个rest.Config的函数
4. restMapperFn：用于获取一个restMapper的函数
5. objectTyper：objectTyper，用于获取一个资源对象的GVK
6. negotiatedSerializer：协商序列化器
7. requireObject：构建出的Result是否必须有资源对象在Info中

## 构建函数
```go
// k8s.io/cli-runtime/pkg/resource/builder.go
// 接收一个RESTClientGetter，创建一个Builder
func NewBuilder(restClientGetter RESTClientGetter) *Builder {
	categoryExpanderFn := func() (restmapper.CategoryExpander, error) {
		discoveryClient, err := restClientGetter.ToDiscoveryClient()
		if err != nil {
			return nil, err
		}
		return restmapper.NewDiscoveryCategoryExpander(discoveryClient), err
	}

	return newBuilder(
		// 新建一个Builder，主要是将clientConfigFn，restMapperFn，categoryExpanderFn三个字段赋值
		restClientGetter.ToRESTConfig,
		restClientGetter.ToRESTMapper,
		(&cachingCategoryExpanderFunc{delegate: categoryExpanderFn}).ToCategoryExpander,
	)
}

func newBuilder(clientConfigFn ClientConfigFunc, restMapper RESTMapperFunc, categoryExpander CategoryExpanderFunc) *Builder {
	return &Builder{
		clientConfigFn:     clientConfigFn,
		restMapperFn:       restMapper,
		categoryExpanderFn: categoryExpander,
		requireObject:      true,
	}
}
```

## Unstructured
```go
// k8s.io/cli-runtime/pkg/resource/builder.go
// 使用Unstructured来序列化对象
func (b *Builder) Unstructured() *Builder {
	if b.mapper != nil {
		b.errs = append(b.errs, fmt.Errorf("another mapper was already selected, cannot use unstructured types"))
		return b
	}
	b.objectTyper = unstructuredscheme.NewUnstructuredObjectTyper()
	b.mapper = &mapper{
		localFn:      b.isLocal,
		restMapperFn: b.restMapperFn,
		clientFn:     b.getClient,
		// 解码器使用unstructured解码器
		decoder:      &metadataValidatingDecoder{unstructured.UnstructuredJSONScheme},
	}

	return b
}
```

## ResourceTypeOrNameArgs
```go
// k8s.io/cli-runtime/pkg/resource/builder.go
func (b *Builder) ResourceTypeOrNameArgs(allowEmptySelector bool, args ...string) *Builder {
	args = normalizeMultipleResourcesArgs(args)
	if ok, err := hasCombinedTypeArgs(args); ok {
		if err != nil {
			b.errs = append(b.errs, err)
			return b
		}
		for _, s := range args {
			tuple, ok, err := splitResourceTypeName(s)
			if err != nil {
				b.errs = append(b.errs, err)
				return b
			}
			if ok {
				b.resourceTuples = append(b.resourceTuples, tuple)
			}
		}
		return b
	}
	if len(args) > 0 {
		// Try replacing aliases only in types
		args[0] = b.ReplaceAliases(args[0])
	}
	switch {
	case len(args) > 2:
		b.names = append(b.names, args[1:]...)
		b.ResourceTypes(SplitResourceArgument(args[0])...)
	case len(args) == 2:
		b.names = append(b.names, args[1])
		b.ResourceTypes(SplitResourceArgument(args[0])...)
	case len(args) == 1:
		b.ResourceTypes(SplitResourceArgument(args[0])...)
		if b.labelSelector == nil && allowEmptySelector {
			selector := labels.Everything().String()
			b.labelSelector = &selector
		}
	case len(args) == 0:
	default:
		b.errs = append(b.errs, fmt.Errorf("arguments must consist of a resource or a resource and name"))
	}
	return b
}
```