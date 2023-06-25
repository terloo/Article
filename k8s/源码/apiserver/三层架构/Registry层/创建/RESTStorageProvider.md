# RESTStorageProvider
RESTStorageProvider是KubeAPIServer定义的用于各个Gourp返回其对应的GroupInfo的接口  
构建APIGroupInfo时会构建Registry层和Storage层

## 接口
```go
// pkg/controlplane/instance.go
type RESTStorageProvider interface {
	GroupName() string
	NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, bool, error)
}
```
1. GroupName：返回组名
2. NewRESTStorage：传入一个RESTOptionsGetter来构建该组所有版本所有资源的Registry层和Srtoage层

## 实现类
以apps为例，由于是便于方法调用的实现类，所以没有字段，仅实现方法
```go
// pkg/registry/apps/rest/storage_apps.go
type StorageProvider struct{}
```

## NewRESTStorage
```go
// pkg/registry/apps/rest/storage_apps.go
func (p StorageProvider) NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, bool, error) {
    // 创建一个apiGroupInfo
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apps.GroupName, legacyscheme.Scheme, legacyscheme.ParameterCodec, legacyscheme.Codecs)
	// If you add a version here, be sure to add an entry in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go with specific priorities.
	// TODO refactor the plumbing to provide the information in the APIGroupInfo

	if apiResourceConfigSource.VersionEnabled(appsapiv1.SchemeGroupVersion) {
        // 调用RStorage方法来创建该版本中所有资源的Storage层
		storageMap, err := p.v1Storage(apiResourceConfigSource, restOptionsGetter)
		if err != nil {
			return genericapiserver.APIGroupInfo{}, false, err
		}
        // 按版本分类放入apiGroupInfo中保存
		apiGroupInfo.VersionedResourcesStorageMap[appsapiv1.SchemeGroupVersion.Version] = storageMap
	}

	return apiGroupInfo, true, nil
}
```

## VStorage
名为`<Version>Storage`的方法，在其中调用Resource的NewStorage方法
```go
// pkg/registry/apps/rest/storage_apps.go
func (p StorageProvider) v1Storage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (map[string]rest.Storage, error) {
    // 构建了一个map用于存储所有资源的Storage层
	storage := map[string]rest.Storage{}

	// deployments
    // 调用资源对应的NewStorage构造资源及其子资源的Storage层
	deploymentStorage, err := deploymentstore.NewStorage(restOptionsGetter)
	if err != nil {
		return storage, err
	}
    // 将该资源对应的所有Storage层存储到map中
	storage["deployments"] = deploymentStorage.Deployment
	storage["deployments/status"] = deploymentStorage.Status
	storage["deployments/scale"] = deploymentStorage.Scale

	// statefulsets
	statefulSetStorage, err := statefulsetstore.NewStorage(restOptionsGetter)
	if err != nil {
		return storage, err
	}
	storage["statefulsets"] = statefulSetStorage.StatefulSet
	storage["statefulsets/status"] = statefulSetStorage.Status
	storage["statefulsets/scale"] = statefulSetStorage.Scale

	// daemonsets
	daemonSetStorage, daemonSetStatusStorage, err := daemonsetstore.NewREST(restOptionsGetter)
	if err != nil {
		return storage, err
	}
	storage["daemonsets"] = daemonSetStorage
	storage["daemonsets/status"] = daemonSetStatusStorage

	// replicasets
	replicaSetStorage, err := replicasetstore.NewStorage(restOptionsGetter)
	if err != nil {
		return storage, err
	}
	storage["replicasets"] = replicaSetStorage.ReplicaSet
	storage["replicasets/status"] = replicaSetStorage.Status
	storage["replicasets/scale"] = replicaSetStorage.Scale

	// controllerrevisions
	historyStorage, err := controllerrevisionsstore.NewREST(restOptionsGetter)
	if err != nil {
		return storage, err
	}
	storage["controllerrevisions"] = historyStorage

	return storage, nil
}
```
