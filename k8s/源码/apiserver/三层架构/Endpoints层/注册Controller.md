# 注册Controller
将APIGroupInfo中保存的Service层信息，绑定到go-restful的webService中，作为Controller层

## 概述
k8s通过两个方法来注册Controller
1. GenericAPIServer.InstallLegacyAPIGroup：用于注册核心库资源`/api/v1/<resource>`
2. GenericAPIServer.InstallAPIGroup：用于注册非核心库资源`/apis/<gourp>/<version>/<resource>`
3. 底层都是调用`GenericAPIServer.installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, openAPIModels openapiproto.Models) error`

## GenericAPIServer.installAPIResources
```go
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, openAPIModels openapiproto.Models) error {
	var resourceInfos []*storageversion.ResourceInfo
	for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
        // 按优先级进行Controller层注册
		if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
			klog.Warningf("Skipping API %v because it has no resources.", groupVersion)
			continue
		}

        // 先获取apiGroupVersion
		apiGroupVersion := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)
		if apiGroupInfo.OptionsExternalVersion != nil {
			apiGroupVersion.OptionsExternalVersion = apiGroupInfo.OptionsExternalVersion
		}
		apiGroupVersion.OpenAPIModels = openAPIModels

		if openAPIModels != nil && utilfeature.DefaultFeatureGate.Enabled(features.ServerSideApply) {
			typeConverter, err := fieldmanager.NewTypeConverter(openAPIModels, false)
			if err != nil {
				return err
			}
			apiGroupVersion.TypeConverter = typeConverter
		}

		apiGroupVersion.MaxRequestBodyBytes = s.maxRequestBodyBytes

        // 调用apiGroupVersion的InstallREST，传入go-restful容器进行注册
		// 在注册APIGroupInfo里的接口时，都是注册到GoRestfulContainer中，而不是NonGoRestfulMux
		r, err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
		if err != nil {
			return fmt.Errorf("unable to setup API %v: %v", apiGroupInfo, err)
		}
		resourceInfos = append(resourceInfos, r...)
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.StorageVersionAPI) &&
		utilfeature.DefaultFeatureGate.Enabled(features.APIServerIdentity) {
		// API installation happens before we start listening on the handlers,
		// therefore it is safe to register ResourceInfos here. The handler will block
		// write requests until the storage versions of the targeting resources are updated.
		s.StorageVersionManager.AddResourceInfo(resourceInfos...)
	}

	return nil
}
```
