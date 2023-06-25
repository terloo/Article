# StandardStorage
StandardStorage是k8s Resgisty层用于增删改查的接口。分为增删改查Watch接口，和策略接口。

### 增删改查接口
```go
// vendor/k8s.io/pkg/apiserver/registry/rest/rest.go
// 标准Service接口，集合了所有公用的操作
type StandardStorage interface {
	Getter
	Lister
	CreaterUpdater
	GracefulDeleter
	CollectionDeleter
	Watcher
}

// Service层的实现在外部一般表现为此接口
type Storage interface {
	New() runtime.Object
}

type Getter interface {
	Get(ctx context.Context, name string, options *metav1.GetOptions) (runtime.Object, error)
}

type Lister interface {
	NewList() runtime.Object
	List(ctx context.Context, options *metainternalversion.ListOptions) (runtime.Object, error)
	TableConvertor
}

type TableConvertor interface {
	ConvertToTable(ctx context.Context, object runtime.Object, tableOptions runtime.Object) (*metav1.Table, error)
}

type CreaterUpdater interface {
	Creater
	Update(ctx context.Context, name string, objInfo UpdatedObjectInfo, createValidation ValidateObjectFunc, updateValidation ValidateObjectUpdateFunc, forceAllowCreate bool, options *metav1.UpdateOptions) (runtime.Object, bool, error)
}

type Creater interface {
	New() runtime.Object
	Create(ctx context.Context, obj runtime.Object, createValidation ValidateObjectFunc, options *metav1.CreateOptions) (runtime.Object, error)
}

type GracefulDeleter interface {
	Delete(ctx context.Context, name string, deleteValidation ValidateObjectFunc, options *metav1.DeleteOptions) (runtime.Object, bool, error)
}

type CollectionDeleter interface {
	DeleteCollection(ctx context.Context, deleteValidation ValidateObjectFunc, options *metav1.DeleteOptions, listOptions *metainternalversion.ListOptions) (runtime.Object, error)
}

type Watcher interface {
	Watch(ctx context.Context, options *metainternalversion.ListOptions) (watch.Interface, error)
}
```

## 实现类
