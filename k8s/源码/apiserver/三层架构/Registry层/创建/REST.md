# REST
每个GR都会创建用于该GR和GR所有子资源的Store实例，并将其包装到apiGroupInfo中，返回给Controller层使用。

## 结构体
k8s的每种资源的RStorage接口一般定义在`pkg/registry/<Group>/<Resource>/storage/storage.go`中
```go
// 以Deployment为例
// pkg/registry/apps/deployment/storage/storage.go

// 主资源的Registry层的结构体都名为REST
type REST struct {
	// 模板实现类
	*genericregistry.Store
	// 分组，用于命令行get命令
	categories []string
}

type StatusREST struct {
	store *genericregistry.Store
}

type RollbackREST struct {
	store *genericregistry.Store
}

type ScaleREST struct {
	store *genericregistry.Store
}

// <Resource>Storage是主资源以及子资源Registry接口的简单组合，防止NewStorage返回过多的Registry接口
type DeploymentStorage struct {
	Deployment *REST
	Status     *StatusREST
	Scale      *ScaleREST
	Rollback   *RollbackREST
}
```

## 构造方法
```go
// pkg/registry/apis/apps/deployment/storage/storage.go

// NewStorage调用NewREST()获取各个资源的REST结构体并封装为DeploymentStorage
// 传入的实现类是RESTOptions
func NewStorage(optsGetter generic.RESTOptionsGetter) (DeploymentStorage, error) {
	deploymentRest, deploymentStatusRest, deploymentRollbackRest, err := NewREST(optsGetter)
	if err != nil {
		return DeploymentStorage{}, err
	}

	return DeploymentStorage{
		Deployment: deploymentRest,
		Status:     deploymentStatusRest,
		Scale:      &ScaleREST{store: deploymentRest.Store},
		Rollback:   deploymentRollbackRest,
	}, nil
}

func NewREST(optsGetter generic.RESTOptionsGetter) (*REST, *StatusREST, *RollbackREST, error) {
	// 直接构造一个genericregistry.Store，并给策略方法赋值
	store := &genericregistry.Store{
		NewFunc:                  func() runtime.Object { return &apps.Deployment{} },
		NewListFunc:              func() runtime.Object { return &apps.DeploymentList{} },
		DefaultQualifiedResource: apps.Resource("deployments"),

		CreateStrategy:      deployment.Strategy,
		UpdateStrategy:      deployment.Strategy,
		DeleteStrategy:      deployment.Strategy,
		ResetFieldsStrategy: deployment.Strategy,

		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
	}

	// 使用传入的RESTOptions来填充store的其他字段，和Storage层
	options := &generic.StoreOptions{RESTOptions: optsGetter}
	if err := store.CompleteWithOptions(options); err != nil {
		return nil, nil, nil, err
	}

	statusStore := *store
	statusStore.UpdateStrategy = deployment.StatusStrategy
	statusStore.ResetFieldsStrategy = deployment.StatusStrategy
	return &REST{store, []string{"all"}}, &StatusREST{store: &statusStore}, &RollbackREST{store: store}, nil
}
```