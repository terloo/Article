# DeferredDiscoveryRESTMapper
DeferredDiscoveryRESTMapper是client-go中提供的基于Discovery客户端实现的RESTMapper接口实现类

## 实现类
```go
// k8s.io/client-go/restmapper/discovery.go
type DeferredDiscoveryRESTMapper struct {
	initMu   sync.Mutex
	delegate meta.RESTMapper
	cl       discovery.CachedDiscoveryInterface
}
```
1. initMu：初始化锁
2. delegate：代理RESTMapper
3. cl：Discovery客户端，从客户端中查询的数据会被写入到delegate

## 构建函数
由于DefeeredDiscoveryRESTMapper使用了懒加载机制，所以构建时不会请求api-server获取数据
```go
// k8s.io/client-go/restmapper/discovery.go
// 需要接收一个CachedDiscovery客户端
func NewDeferredDiscoveryRESTMapper(cl discovery.CachedDiscoveryInterface) *DeferredDiscoveryRESTMapper {
	return &DeferredDiscoveryRESTMapper{
		cl: cl,
	}
}
```

## GetAPIGroupResources
调用discovery客户端，获取api-resources相关数据
```go
// k8s.io/client-go/restmapper/discovery.go
func GetAPIGroupResources(cl discovery.DiscoveryInterface) ([]*APIGroupResources, error) {
    // 获取所有APIGroup和所有APIResourceList
	gs, rs, err := cl.ServerGroupsAndResources()
	if rs == nil || gs == nil {
		return nil, err
		// TODO track the errors and update callers to handle partial errors.
	}
	rsm := map[string]*metav1.APIResourceList{}
	for _, r := range rs {
		rsm[r.GroupVersion] = r
	}

    // 将[]APIResourceList处理[]APIGroupResources
	var result []*APIGroupResources
	for _, group := range gs {
		groupResources := &APIGroupResources{
			Group:              *group,
			VersionedResources: make(map[string][]metav1.APIResource),
		}
		for _, version := range group.Versions {
			resources, ok := rsm[version.GroupVersion]
			if !ok {
				continue
			}
			groupResources.VersionedResources[version.Version] = resources.APIResources
		}
		result = append(result, groupResources)
	}
	return result, nil
}
```

## getDelegate
初始化代理RESTMapper
```go
// k8s.io/client-go/restmapper/discovery.go
func (d *DeferredDiscoveryRESTMapper) getDelegate() (meta.RESTMapper, error) {
	d.initMu.Lock()
	defer d.initMu.Unlock()

	if d.delegate != nil {
		return d.delegate, nil
	}

	groupResources, err := GetAPIGroupResources(d.cl)
	if err != nil {
		return nil, err
	}

	// 使用[]APIGroupResources来创建DiscoveryRESTMapper
	d.delegate = NewDiscoveryRESTMapper(groupResources)
	return d.delegate, nil
}
```