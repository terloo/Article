# SimpleRestOptionsFactory
SimpleRestOptionsFactory可以构建RESTOptionsGetter，用于创建非KubeAPIService所管理k8s内置资源的Service层，即`apiextensions.k8s.io`和`apiregistration.k8s.io`组的资源
SimpleRestOptionsFactory直接使用命令行中Etcd相关命令行选项来构建RESTOptionsGetter

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/server/options/etcd.go
type SimpleRestOptionsFactory struct {
	Options              EtcdOptions
	TransformerOverrides map[schema.GroupResource]value.Transformer
}
```
1. Options：ETCD相关命令行选项，生成RESTOptions时将会解构该封装
2. TransformerOverrides：解密传输数据的转换器

## GetRESTOptions
```go
// vendor/k8s.io/apiserver/pkg/server/options/etcd.go
func (f *SimpleRestOptionsFactory) GetRESTOptions(resource schema.GroupResource) (generic.RESTOptions, error) {
    // 解构ETCD相关参数封装，并重新封装到RESTOptions中
	ret := generic.RESTOptions{
		StorageConfig:             f.Options.StorageConfig.ForResource(resource),
		Decorator:                 generic.UndecoratedStorage,
		EnableGarbageCollection:   f.Options.EnableGarbageCollection,
		DeleteCollectionWorkers:   f.Options.DeleteCollectionWorkers,
		// Etcd存储路径带上组名
		ResourcePrefix:            resource.Group + "/" + resource.Resource,
		CountMetricPollPeriod:     f.Options.StorageConfig.CountMetricPollPeriod,
		StorageObjectCountTracker: f.Options.StorageConfig.StorageObjectCountTracker,
	}
    // 生成传输数据的加解密转换器
	if f.TransformerOverrides != nil {
		if transformer, ok := f.TransformerOverrides[resource]; ok {
			ret.StorageConfig.Transformer = transformer
		}
	}
    // 处理watch缓存
	if f.Options.EnableWatchCache {
		sizes, err := ParseWatchCacheSizes(f.Options.WatchCacheSizes)
		if err != nil {
			return generic.RESTOptions{}, err
		}
		size, ok := sizes[resource]
		if ok && size > 0 {
			klog.Warningf("Dropping watch-cache-size for %v - watchCache size is now dynamic", resource)
		}
        // 如果不需要缓存，使用UnderlayingStorage
		if ok && size <= 0 {
			ret.Decorator = generic.UndecoratedStorage
		} else {
            // 如果需要缓存，使用CacherStorage
			ret.Decorator = genericregistry.StorageWithCacher()
		}
	}
	return ret, nil
}
```