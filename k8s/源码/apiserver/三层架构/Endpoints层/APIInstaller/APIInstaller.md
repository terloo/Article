# APIInstaller
APIInstaller组合了APIGroupVersion，是专门用于生成go-restful的Route并注册到指定的WebService的结构体

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
type APIInstaller struct {
	group             *APIGroupVersion
	prefix            string // Path prefix where API resources are to be registered.
	minRequestTimeout time.Duration
}
```

## Install()
遍历APIGroupVersion.Storage的key字典排序后，添加到WebService中
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) Install() ([]metav1.APIResource, []*storageversion.ResourceInfo, *restful.WebService, []error) {
	var apiResources []metav1.APIResource
	var resourceInfos []*storageversion.ResourceInfo
	var errors []error
	// 为每个GV创建一个WebService
	ws := a.newWebService()

	// Register the paths in a deterministic (sorted) order to get a deterministic swagger spec.
	// path为Resource类似于deployments deployments/status这种字符串
	paths := make([]string, len(a.group.Storage))
	var i int = 0
	for path := range a.group.Storage {
		paths[i] = path
		i++
	}
	// 将路径排字典序
	sort.Strings(paths)
	for _, path := range paths {
		// 在这个方法里生成Route，注册到WebService
		apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
		if err != nil {
			errors = append(errors, fmt.Errorf("error in registering resource: %s, %v", path, err))
		}
		if apiResource != nil {
			apiResources = append(apiResources, *apiResource)
		}
		if resourceInfo != nil {
			resourceInfos = append(resourceInfos, resourceInfo)
		}
	}
	return apiResources, resourceInfos, ws, errors
}
```

