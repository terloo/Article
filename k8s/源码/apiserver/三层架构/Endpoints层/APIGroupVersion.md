# APIGroupVersion
由APIGroupInfo构建而来，保存着某个GV相关的api根路径等信息，通过此对象来构建APIInstaller

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/endpoints/groupversion.go
type APIGroupVersion struct {
    // GV对应的的所有Service层
	Storage map[string]rest.Storage

    // 根路由
	Root string

	// GroupVersion is the external group version
	GroupVersion schema.GroupVersion

	// OptionsExternalVersion controls the Kubernetes APIVersion used for common objects in the apiserver
	// schema like api.Status, api.DeleteOptions, and metav1.ListOptions. Other implementors may
	// define a version "v1beta1" but want to use the Kubernetes "v1" internal objects. If
	// empty, defaults to GroupVersion.
	OptionsExternalVersion *schema.GroupVersion
	// MetaGroupVersion defaults to "meta.k8s.io/v1" and is the scheme group version used to decode
	// common API implementations like ListOptions. Future changes will allow this to vary by group
	// version (for when the inevitable meta/v2 group emerges).
	MetaGroupVersion *schema.GroupVersion

	// RootScopedKinds are the root scoped kinds for the primary GroupVersion
	RootScopedKinds sets.String

	// Serializer is used to determine how to convert responses from API methods into bytes to send over
	// the wire.
	Serializer     runtime.NegotiatedSerializer
	ParameterCodec runtime.ParameterCodec

	Typer                 runtime.ObjectTyper
	Creater               runtime.ObjectCreater
	Convertor             runtime.ObjectConvertor
	ConvertabilityChecker ConvertabilityChecker
	Defaulter             runtime.ObjectDefaulter
	Linker                runtime.SelfLinker
	UnsafeConvertor       runtime.ObjectConvertor
	TypeConverter         fieldmanager.TypeConverter

	EquivalentResourceRegistry runtime.EquivalentResourceRegistry

	// Authorizer determines whether a user is allowed to make a certain request. The Handler does a preliminary
	// authorization check using the request URI but it may be necessary to make additional checks, such as in
	// the create-on-update case
	Authorizer authorizer.Authorizer

	Admit admission.Interface

	MinRequestTimeout time.Duration

	// OpenAPIModels exposes the OpenAPI models to each individual handler.
	OpenAPIModels openapiproto.Models

	// The limit on the request body size that would be accepted and decoded in a write request.
	// 0 means no limit.
	MaxRequestBodyBytes int64
}
```

## 构造APIGroupVersion
apiGroupVersion在k8s中一般从GenericAPIServer对象构造，需要APIGroupInfo，需要注册的GV，以及该G的apiPrefix
```go
// vendor/k8s.io/pkg/server/genericapiserver.go
func (s *GenericAPIServer) getAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion, apiPrefix string) *genericapi.APIGroupVersion {
	storage := make(map[string]rest.Storage)
    // 将GV对应的所有Service层重新封装
	for k, v := range apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version] {
		storage[strings.ToLower(k)] = v
	}
	version := s.newAPIGroupVersion(apiGroupInfo, groupVersion)
	version.Root = apiPrefix
	version.Storage = storage
	return version
}

func (s *GenericAPIServer) newAPIGroupVersion(apiGroupInfo *APIGroupInfo, groupVersion schema.GroupVersion) *genericapi.APIGroupVersion {
	return &genericapi.APIGroupVersion{
		GroupVersion:     groupVersion,
		MetaGroupVersion: apiGroupInfo.MetaGroupVersion,

		ParameterCodec:        apiGroupInfo.ParameterCodec,
		Serializer:            apiGroupInfo.NegotiatedSerializer,
		Creater:               apiGroupInfo.Scheme,
		Convertor:             apiGroupInfo.Scheme,
		ConvertabilityChecker: apiGroupInfo.Scheme,
		UnsafeConvertor:       runtime.UnsafeObjectConvertor(apiGroupInfo.Scheme),
		Defaulter:             apiGroupInfo.Scheme,
		Typer:                 apiGroupInfo.Scheme,
		Linker:                runtime.SelfLinker(meta.NewAccessor()),

		EquivalentResourceRegistry: s.EquivalentResourceRegistry,

		Admit:             s.admissionControl,
		MinRequestTimeout: s.minRequestTimeout,
		Authorizer:        s.Authorizer,
	}
}
```

## InstallRest注册Service到Container
从APIGroupVersion中构建出APIInstaller，调用APIInstaller的方法生成该GV对应的WebService，将WebService注册到container中
```go
// vendor/k8s.io/apiserver/endpoints/groupversion.go
func (g *APIGroupVersion) InstallREST(container *restful.Container) ([]*storageversion.ResourceInfo, error) {
    // 拼接Controller层路径前半部分
    // <rootPath>/<group>/<version>
	prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
	installer := &APIInstaller{
		group:             g, // APIInstaller组合了APiGroupVersion
		prefix:            prefix,
		minRequestTimeout: g.MinRequestTimeout,
	}

    // 调用构造出来的APIInstaller.Install()
    // 构造go-restful 资源相关webService，并将生成Route与该webService绑定
	apiResources, resourceInfos, ws, registrationErrors := installer.Install()
	versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
    // 将Discovery相关路由与WebService绑定
	versionDiscoveryHandler.AddToWebService(ws)
    // 将WebService添加到container
	container.Add(ws)
	return removeNonPersistedResources(resourceInfos), utilerrors.NewAggregate(registrationErrors)
}
```