# StorageFactoryRestOptionsFactory
StorageFactoryRestOptionsFactory可以构建RESTOptionsGetter，用于创建KubeAPIService所管理k8s内置资源的Service层。
StorageFactoryRestOptionsFactory使用了StorageFactory用于构建RESTOptions，所以可以为不同的资源生成不同的Storage层配置类

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/server/options/etcd.go
type StorageFactoryRestOptionsFactory struct {
	Options        EtcdOptions
	StorageFactory serverstorage.StorageFactory
}
```

## GetRESTOptions
```go
// vendor/k8s.io/apiserver/pkg/server/options/etcd.go
func (f *StorageFactoryRestOptionsFactory) GetRESTOptions(resource schema.GroupResource) (generic.RESTOptions, error) {
    // 调用StorageFactory的NewConfig方法来获取storageConfig
	storageConfig, err := f.StorageFactory.NewConfig(resource)
	if err != nil {
		return generic.RESTOptions{}, fmt.Errorf("unable to find storage destination for %v, due to %v", resource, err.Error())
	}

	ret := generic.RESTOptions{
		StorageConfig:             storageConfig,
		Decorator:                 generic.UndecoratedStorage,
		DeleteCollectionWorkers:   f.Options.DeleteCollectionWorkers,
		EnableGarbageCollection:   f.Options.EnableGarbageCollection,
		// 资源的存储前缀亦通过StorageFactory提供的方法来生成
		ResourcePrefix:            f.StorageFactory.ResourcePrefix(resource),
		CountMetricPollPeriod:     f.Options.StorageConfig.CountMetricPollPeriod,
		StorageObjectCountTracker: f.Options.StorageConfig.StorageObjectCountTracker,
	}
	if f.Options.EnableWatchCache {
		sizes, err := ParseWatchCacheSizes(f.Options.WatchCacheSizes)
		if err != nil {
			return generic.RESTOptions{}, err
		}
		size, ok := sizes[resource]
		if ok && size > 0 {
			klog.Warningf("Dropping watch-cache-size for %v - watchCache size is now dynamic", resource)
		}
		if ok && size <= 0 {
			ret.Decorator = generic.UndecoratedStorage
		} else {
			ret.Decorator = genericregistry.StorageWithCacher()
		}
	}

	return ret, nil
}
```