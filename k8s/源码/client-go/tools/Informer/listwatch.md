# listwatch
Reflect使用此接口的实现类与api-server交互获取资源对象

## 接口
```go
// vendor/k8s.io/client-go/tools/cache/listwatch.go
// ListerWatcher：
type Lister interface {
    // 返回xxxList资源对象
	List(options metav1.ListOptions) (runtime.Object, error)
}

type Watcher interface {
    // 对指定的版本开启一次watch
	Watch(options metav1.ListOptions) (watch.Interface, error)
}

type ListerWatcher interface {
	Lister
	Watcher
}
```

## 实现类
此实现类是模板实现，主要功能由其中的ListFunc和WatchFunc进行实现
```go
// vendor/k8s.io/client-go/tools/cache/listwatch.go
type ListWatch struct {
	ListFunc  ListFunc
	WatchFunc WatchFunc
	// DisableChunking requests no chunking for this list watcher.
	DisableChunking bool
}

// 传入一个ListOptions进行List操作
type ListFunc func(options metav1.ListOptions) (runtime.Object, error)

// 传入一个ListOptions进行Watch操作
type WatchFunc func(options metav1.ListOptions) (watch.Interface, error)
```

## 构造函数
```go
// vendor/k8s.io/client-go/tools/cache/listwatch.go
// Get()返回restclient.Request对象，一般直接传入RESTClient实现类
type Getter interface {
	Get() *restclient.Request
}

// NewListWatchFromClient creates a new ListWatch from the specified client, resource, namespace and field selector.
func NewListWatchFromClient(c Getter, resource string, namespace string, fieldSelector fields.Selector) *ListWatch {
	optionsModifier := func(options *metav1.ListOptions) {
		options.FieldSelector = fieldSelector.String()
	}
	return NewFilteredListWatchFromClient(c, resource, namespace, optionsModifier)
}

// NewFilteredListWatchFromClient creates a new ListWatch from the specified client, resource, namespace, and option modifier.
// Option modifier is a function takes a ListOptions and modifies the consumed ListOptions. Provide customized modifier function
// to apply modification to ListOptions with a field selector, a label selector, or any other desired options.
func NewFilteredListWatchFromClient(c Getter, resource string, namespace string, optionsModifier func(options *metav1.ListOptions)) *ListWatch {
    // 将List操作包装为listFunc
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		optionsModifier(&options)
		return c.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			Do(context.TODO()).
			Get()
	}
    // 将Watch操作包装为watchFunc
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.Watch = true
		optionsModifier(&options)
		return c.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			Watch(context.TODO())
	}
	return &ListWatch{ListFunc: listFunc, WatchFunc: watchFunc}
}

// List a set of apiserver resources
func (lw *ListWatch) List(options metav1.ListOptions) (runtime.Object, error) {
	// ListWatch is used in Reflector, which already supports pagination.
	// Don't paginate here to avoid duplication.
	return lw.ListFunc(options)
}

// Watch a set of apiserver resources
func (lw *ListWatch) Watch(options metav1.ListOptions) (watch.Interface, error) {
	return lw.WatchFunc(options)
}
```