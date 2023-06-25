# StorageFactoryConfig
StorageFactoryConfig是用于构建StorageFactory的配置类

## 结构体
```go
// pkg/kubeapiserver/default_storage_factory_builder.go
type StorageFactoryConfig struct {
	StorageConfig                    storagebackend.Config
	APIResourceConfig                *serverstorage.ResourceConfig
	DefaultResourceEncoding          *serverstorage.DefaultResourceEncodingConfig
	DefaultStorageMediaType          string
	Serializer                       runtime.StorageSerializer
	ResourceEncodingOverrides        []schema.GroupVersionResource
	EtcdServersOverrides             []string
	EncryptionProviderConfigFilepath string
}
```

## 构造函数
其余字段在Complete函数中赋值
```go
// pkg/kubeapiserver/default_storage_factory_builder.go
func NewStorageFactoryConfig() *StorageFactoryConfig {

    // 有一个资源对象需要指定特殊的Encoding
	resources := []schema.GroupVersionResource{
		apisstorage.Resource("csistoragecapacities").WithVersion("v1beta1"),
	}

	return &StorageFactoryConfig{
		Serializer:                legacyscheme.Codecs,
		DefaultResourceEncoding:   serverstorage.NewDefaultResourceEncodingConfig(legacyscheme.Scheme),
		ResourceEncodingOverrides: resources,
	}
}
```

## Complete
Complete接收一个etcdOptions，将其中部分字段赋值到自身，返回一个completedStorageFactoryConfig
```go
// pkg/kubeapiserver/default_storage_factory_builder.go
func (c *StorageFactoryConfig) Complete(etcdOptions *serveroptions.EtcdOptions) (*completedStorageFactoryConfig, error) {
	c.StorageConfig = etcdOptions.StorageConfig
	c.DefaultStorageMediaType = etcdOptions.DefaultStorageMediaType
	c.EtcdServersOverrides = etcdOptions.EtcdServersOverrides
	c.EncryptionProviderConfigFilepath = etcdOptions.EncryptionProviderConfigFilepath
	return &completedStorageFactoryConfig{c}, nil
}
```

## New
通过Config来构建StorageFacotry
```go
// pkg/kubeapiserver/default_storage_factory_builder.go
func (c *completedStorageFactoryConfig) New() (*serverstorage.DefaultStorageFactory, error) {
	resourceEncodingConfig := resourceconfig.MergeResourceEncodingConfigs(c.DefaultResourceEncoding, c.ResourceEncodingOverrides)
    // 构建
	storageFactory := serverstorage.NewDefaultStorageFactory(
        // ETCD相关参数
		c.StorageConfig,
        // 默认存储数据格式
		c.DefaultStorageMediaType,
		c.Serializer,
		resourceEncodingConfig,
        // GV的开启与关闭配置
		c.APIResourceConfig,
        // 一些资源的特殊前缀
		SpecialDefaultResourcePrefixes)

	storageFactory.AddCohabitatingResources(networking.Resource("networkpolicies"), extensions.Resource("networkpolicies"))
	storageFactory.AddCohabitatingResources(apps.Resource("deployments"), extensions.Resource("deployments"))
	storageFactory.AddCohabitatingResources(apps.Resource("daemonsets"), extensions.Resource("daemonsets"))
	storageFactory.AddCohabitatingResources(apps.Resource("replicasets"), extensions.Resource("replicasets"))
	storageFactory.AddCohabitatingResources(api.Resource("events"), events.Resource("events"))
	storageFactory.AddCohabitatingResources(api.Resource("replicationcontrollers"), extensions.Resource("replicationcontrollers")) // to make scale subresources equivalent
	storageFactory.AddCohabitatingResources(policy.Resource("podsecuritypolicies"), extensions.Resource("podsecuritypolicies"))
	storageFactory.AddCohabitatingResources(networking.Resource("ingresses"), extensions.Resource("ingresses"))

	for _, override := range c.EtcdServersOverrides {
		tokens := strings.Split(override, "#")
		apiresource := strings.Split(tokens[0], "/")

		group := apiresource[0]
		resource := apiresource[1]
		groupResource := schema.GroupResource{Group: group, Resource: resource}

		servers := strings.Split(tokens[1], ";")
		storageFactory.SetEtcdLocation(groupResource, servers)
	}
	if len(c.EncryptionProviderConfigFilepath) != 0 {
		transformerOverrides, err := encryptionconfig.GetTransformerOverrides(c.EncryptionProviderConfigFilepath)
		if err != nil {
			return nil, err
		}
		for groupResource, transformer := range transformerOverrides {
			storageFactory.SetTransformer(groupResource, transformer)
		}
	}
	return storageFactory, nil
}

// 一些资源对象拥有特殊的存储前缀
var SpecialDefaultResourcePrefixes = map[schema.GroupResource]string{
	{Group: "", Resource: "replicationcontrollers"}:        "controllers",
	{Group: "", Resource: "endpoints"}:                     "services/endpoints",
	{Group: "", Resource: "nodes"}:                         "minions",
	{Group: "", Resource: "services"}:                      "services/specs",
	{Group: "extensions", Resource: "ingresses"}:           "ingress",
	{Group: "networking.k8s.io", Resource: "ingresses"}:    "ingress",
	{Group: "extensions", Resource: "podsecuritypolicies"}: "podsecuritypolicy",
	{Group: "policy", Resource: "podsecuritypolicies"}:     "podsecuritypolicy",
}
```