# CRDRESTOptionsGetter
构建RESTOptionsGetter，用于创建CRD资源的Registry层

## 结构体
```go
// vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_handler.go
type CRDRESTOptionsGetter struct {
	StorageConfig             storagebackend.Config
	StoragePrefix             string
	EnableWatchCache          bool
	DefaultWatchCacheSize     int
	EnableGarbageCollection   bool
	DeleteCollectionWorkers   int
	CountMetricPollPeriod     time.Duration
	StorageObjectCountTracker flowcontrolrequest.StorageObjectCountTracker
}
```
1. 所有的参数都是ETCD相关命令行选项的再封装以及修改部分选项

## 构造函数
与结构体不在一个包中
```go
// vendor/k8s.io/apiextensions-apiserver/pkg/cmd/server/options/options.go
// 传入Etcd的命令行选项
func NewCRDRESTOptionsGetter(etcdOptions genericoptions.EtcdOptions) genericregistry.RESTOptionsGetter {
	ret := apiserver.CRDRESTOptionsGetter{
		StorageConfig:             etcdOptions.StorageConfig,
		StoragePrefix:             etcdOptions.StorageConfig.Prefix,
		EnableWatchCache:          etcdOptions.EnableWatchCache,
		DefaultWatchCacheSize:     etcdOptions.DefaultWatchCacheSize,
		EnableGarbageCollection:   etcdOptions.EnableGarbageCollection,
		DeleteCollectionWorkers:   etcdOptions.DeleteCollectionWorkers,
		CountMetricPollPeriod:     etcdOptions.StorageConfig.CountMetricPollPeriod,
		StorageObjectCountTracker: etcdOptions.StorageConfig.StorageObjectCountTracker,
	}
	ret.StorageConfig.Codec = unstructured.UnstructuredJSONScheme

	return ret
}
```

## GetRESTOptions
```go
// vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_handler.go
func (t CRDRESTOptionsGetter) GetRESTOptions(resource schema.GroupResource) (generic.RESTOptions, error) {
	ret := generic.RESTOptions{
		StorageConfig:             t.StorageConfig.ForResource(resource),
		Decorator:                 generic.UndecoratedStorage,
		EnableGarbageCollection:   t.EnableGarbageCollection,
		DeleteCollectionWorkers:   t.DeleteCollectionWorkers,
        // Etcd中存储路径带上组名
		ResourcePrefix:            resource.Group + "/" + resource.Resource,
		CountMetricPollPeriod:     t.CountMetricPollPeriod,
		StorageObjectCountTracker: t.StorageObjectCountTracker,
	}
    // 如果需要缓存，使用CacherStorage
	if t.EnableWatchCache {
		ret.Decorator = genericregistry.StorageWithCacher()
	}
	return ret, nil
}
```