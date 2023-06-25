# StorageFactory
StorageFactory是StorageFactoryRestOptionsFactory结构体中用于构建RESTOptions的核心结构

## 接口
```go
// vendor/k8s.io/apiserver/pkg/server/storage/storage_factory.go
type StorageFactory interface {
	// New finds the storage destination for the given group and resource. It will
	// return an error if the group has no storage destination configured.
	NewConfig(groupResource schema.GroupResource) (*storagebackend.ConfigForResource, error)

	// ResourcePrefix returns the overridden resource prefix for the GroupResource
	// This allows for cohabitation of resources with different native types and provides
	// centralized control over the shape of etcd directories
	ResourcePrefix(groupResource schema.GroupResource) string

	// Backends gets all backends for all registered storage destinations.
	// Used for getting all instances for health validations.
	Backends() []Backend
}
```
1. NewConfig()：获取一个针对该GR的ConfigForResource
2. ResourcePrefix：获取该GV存储时的前缀路径
3. Backends：获取存储数据库的地址，用于存活检测

## 实现类
```go
// vendor/k8s.io/apiserver/pkg/server/storage/storage_factory.go
type DefaultStorageFactory struct {
	// StorageConfig describes how to create a storage backend in general.
	// Its authentication information will be used for every storage.Interface returned.
	StorageConfig storagebackend.Config

	Overrides map[schema.GroupResource]groupResourceOverrides

	DefaultResourcePrefixes map[schema.GroupResource]string

	// DefaultMediaType is the media type used to store resources. If it is not set, "application/json" is used.
	DefaultMediaType string

	// DefaultSerializer is used to create encoders and decoders for the storage.Interface.
	DefaultSerializer runtime.StorageSerializer

	// ResourceEncodingConfig describes how to encode a particular GroupVersionResource
	ResourceEncodingConfig ResourceEncodingConfig

	// APIResourceConfigSource indicates whether the *storage* is enabled, NOT the API
	// This is discrete from resource enablement because those are separate concerns.  How this source is configured
	// is left to the caller.
	APIResourceConfigSource APIResourceConfigSource

	// newStorageCodecFn exists to be overwritten for unit testing.
	newStorageCodecFn func(opts StorageCodecConfig) (codec runtime.Codec, encodeVersioner runtime.GroupVersioner, err error)
}
```
1. StorageConfig：通用的StorageConfig
2. Overrides：针对特殊的GR，可以将其配置在指定的存储数据库中
3. ResourceEncodingConfig：针对特殊的GR，可以指定该GV在encode时所使用的内外部版本
4. APIResourceConfigSource：用于保存哪些GV启用，哪些GV不启用
5. newStorageCodecFn：新建Codec的函数，用函数的形式保存方便测试

## NewConfig
获取一个针对该GR的ConfigForResource
```go
// vendor/k8s.io/pkg/server/storage/storage_factory.go
func (s *DefaultStorageFactory) NewConfig(groupResource schema.GroupResource) (*storagebackend.ConfigForResource, error) {
	chosenStorageResource := s.getStorageGroupResource(groupResource)

	// operate on copy
	// 先复制一份默认的StorageConfig
	storageConfig := s.StorageConfig
	codecConfig := StorageCodecConfig{
		StorageMediaType:  s.DefaultMediaType,
		StorageSerializer: s.DefaultSerializer,
	}

	// 应用特殊配置
	// 该GV是否有对group/*的特殊Storage配置
	if override, ok := s.Overrides[getAllResourcesAlias(chosenStorageResource)]; ok {
		override.Apply(&storageConfig, &codecConfig)
	}
	// 该GV是否有对group/resource的特殊Storage配置
	if override, ok := s.Overrides[chosenStorageResource]; ok {
		override.Apply(&storageConfig, &codecConfig)
	}

	var err error
	// 使用优先级最高的资源版本作为Encoder的默认编码版本
	codecConfig.StorageVersion, err = s.ResourceEncodingConfig.StorageEncodingFor(chosenStorageResource)
	if err != nil {
		return nil, err
	}
	// 使用内部版本作为Decoder的默认解码版本
	codecConfig.MemoryVersion, err = s.ResourceEncodingConfig.InMemoryEncodingFor(groupResource)
	if err != nil {
		return nil, err
	}
	codecConfig.Config = storageConfig

	// 使用codecConfig来获取codec和Encode版本
	storageConfig.Codec, storageConfig.EncodeVersioner, err = s.newStorageCodecFn(codecConfig)
	if err != nil {
		return nil, err
	}
	klog.V(3).Infof("storing %v in %v, reading as %v from %#v", groupResource, codecConfig.StorageVersion, codecConfig.MemoryVersion, codecConfig.Config)

	return storageConfig.ForResource(groupResource), nil
}
```

## ResourcePrefix
获取该GV存储时的前缀路径
```go
func (s *DefaultStorageFactory) ResourcePrefix(groupResource schema.GroupResource) string {
	chosenStorageResource := s.getStorageGroupResource(groupResource)
	// groupOverride和exactResourceOverride一般情况下都没有东西
	groupOverride := s.Overrides[getAllResourcesAlias(chosenStorageResource)]
	exactResourceOverride := s.Overrides[chosenStorageResource]

	// 特殊前缀，如node得前缀是minions
	etcdResourcePrefix := s.DefaultResourcePrefixes[chosenStorageResource]
	if len(groupOverride.etcdResourcePrefix) > 0 {
		etcdResourcePrefix = groupOverride.etcdResourcePrefix
	}
	if len(exactResourceOverride.etcdResourcePrefix) > 0 {
		etcdResourcePrefix = exactResourceOverride.etcdResourcePrefix
	}
	if len(etcdResourcePrefix) == 0 {
		etcdResourcePrefix = strings.ToLower(chosenStorageResource.Resource)
	}

	return etcdResourcePrefix
}
```