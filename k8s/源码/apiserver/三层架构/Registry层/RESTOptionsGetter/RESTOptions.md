# RESTOptions
RESTOptions用于构建Storage层的配置类，在Service层中构造

## 结构体
```go
// vendor/k8s.io/aiserver/pkg/registry/generic/options.go
type RESTOptions struct {
	StorageConfig *storagebackend.ConfigForResource
	Decorator     StorageDecorator

	EnableGarbageCollection   bool
	DeleteCollectionWorkers   int
	ResourcePrefix            string
	CountMetricPollPeriod     time.Duration
	StorageObjectCountTracker flowcontrolrequest.StorageObjectCountTracker
}
```
1. StorageConfig：为ConfigForResource结构体，此结构体用于为指定的资源创建Storage层
2. Decorator：生成Storage层的函数，分为带缓存和不带缓存
