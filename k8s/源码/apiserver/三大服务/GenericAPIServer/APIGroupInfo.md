# APIGroupInfo
APIGroupInfo是GenericAPIServer提供的用于保存资源Group所有Regsitry层的包装类，GenericAPIServer通过此对象来获取Endpoints层的APIGroupVersion

## 概述
APIGroupInfo保存了每个Group中所有的资源及其对应的资源RESTStorage。在API对象进行注册时，都会被组织成为APIGroupInfo的形式，里面包含了所有注册需要的信息。  
每一个Group都会实例化为一个APIGroupInfo。  
这个结构体仅用于存储，没有方法

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/server/genericapiserver.go
type APIGroupInfo struct {
    // 按优先级排列的该Group下所有GV
	PrioritizedVersions []schema.GroupVersion
	// Info about the resources in this group. It's a map from version to resource to the storage.
    // 保存所有资源对象的RESTStorage(Serverc层)
    // 版本号 -> (资源名字 -> RESTStorage)
	VersionedResourcesStorageMap map[string]map[string]rest.Storage
	// OptionsExternalVersion controls the APIVersion used for common objects in the
	// schema like api.Status, api.DeleteOptions, and metav1.ListOptions. Other implementors may
	// define a version "v1beta1" but want to use the Kubernetes "v1" internal objects.
	// If nil, defaults to groupMeta.GroupVersion.
	// TODO: Remove this when https://github.com/kubernetes/kubernetes/issues/19018 is fixed.
    // 已过时
	OptionsExternalVersion *schema.GroupVersion
	// MetaGroupVersion defaults to "meta.k8s.io/v1" and is the scheme group version used to decode
	// common API implementations like ListOptions. Future changes will allow this to vary by group
	// version (for when the inevitable meta/v2 group emerges).
    // 元数据版本
	MetaGroupVersion *schema.GroupVersion

	// Scheme includes all of the types used by this group and how to convert between them (or
	// to convert objects from outside of this group that are accepted in this API).
	// TODO: replace with interfaces
	Scheme *runtime.Scheme
	// NegotiatedSerializer controls how this group encodes and decodes data
    // 序列化器
	NegotiatedSerializer runtime.NegotiatedSerializer
	// ParameterCodec performs conversions for query parameters passed to API calls
    // 参数序列化器
	ParameterCodec runtime.ParameterCodec

	// StaticOpenAPISpec is the spec derived from the definitions of all resources installed together.
	// It is set during InstallAPIGroups, InstallAPIGroup, and InstallLegacyAPIGroup.
	StaticOpenAPISpec *spec.Swagger
}
```

## 默认构造函数
```go
func NewDefaultAPIGroupInfo(group string, scheme *runtime.Scheme, parameterCodec runtime.ParameterCodec, codecs serializer.CodecFactory) APIGroupInfo {
	return APIGroupInfo{
		PrioritizedVersions:          scheme.PrioritizedVersionsForGroup(group),
		VersionedResourcesStorageMap: map[string]map[string]rest.Storage{},
		// TODO unhardcode this.  It was hardcoded before, but we need to re-evaluate
		OptionsExternalVersion: &schema.GroupVersion{Version: "v1"},
		Scheme:                 scheme,
		ParameterCodec:         parameterCodec,
		NegotiatedSerializer:   codecs,
	}
}
```