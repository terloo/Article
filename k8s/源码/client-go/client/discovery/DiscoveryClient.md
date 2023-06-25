# DiscoveryClient
基础实现，与api-server交互的基本Discovery客户端

## 结构体
```go
// k8s.io/client-go/discovery/discovery_client.go
type DiscoveryClient struct {
	restClient restclient.Interface

	LegacyPrefix string
}
```

## 构造函数
```go
// k8s.io/client-go/discovery/discovery_client.go
// NewDiscoveryClientForConfig creates a new DiscoveryClient for the given config. This client
// can be used to discover supported resources in the API server.
// NewDiscoveryClientForConfig is equivalent to NewDiscoveryClientForConfigAndClient(c, httpClient),
// where httpClient was generated with rest.HTTPClientFor(c).
func NewDiscoveryClientForConfig(c *restclient.Config) (*DiscoveryClient, error) {
	config := *c
	if err := setDiscoveryDefaults(&config); err != nil {
		return nil, err
	}
	httpClient, err := restclient.HTTPClientFor(&config)
	if err != nil {
		return nil, err
	}
	return NewDiscoveryClientForConfigAndClient(&config, httpClient)
}

// NewDiscoveryClientForConfigAndClient creates a new DiscoveryClient for the given config. This client
// can be used to discover supported resources in the API server.
// Note the http client provided takes precedence over the configured transport values.
func NewDiscoveryClientForConfigAndClient(c *restclient.Config, httpClient *http.Client) (*DiscoveryClient, error) {
	config := *c
	if err := setDiscoveryDefaults(&config); err != nil {
		return nil, err
	}
	client, err := restclient.UnversionedRESTClientForConfigAndClient(&config, httpClient)
	return &DiscoveryClient{restClient: client, LegacyPrefix: "/api"}, err
}

// NewDiscoveryClientForConfigOrDie creates a new DiscoveryClient for the given config. If
// there is an error, it panics.
func NewDiscoveryClientForConfigOrDie(c *restclient.Config) *DiscoveryClient {
	client, err := NewDiscoveryClientForConfig(c)
	if err != nil {
		panic(err)
	}
	return client

}

// NewDiscoveryClient returns a new DiscoveryClient for the given RESTClient.
func NewDiscoveryClient(c restclient.Interface) *DiscoveryClient {
	return &DiscoveryClient{restClient: c, LegacyPrefix: "/api"}
}
```

## 方法
```go
// vendor/k8s.io/client-go/discovery/discovery_client.go
func (d *DiscoveryClient) ServerGroups() (apiGroupList *metav1.APIGroupList, err error) {
	// Get the groupVersions exposed at /api
    // 获取/api下面的所有Group
	v := &metav1.APIVersions{}
	err = d.restClient.Get().AbsPath(d.LegacyPrefix).Do(context.TODO()).Into(v)
	apiGroup := metav1.APIGroup{}
	if err == nil && len(v.Versions) != 0 {
		apiGroup = apiVersionsToAPIGroup(v)
	}
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}

	// Get the groupVersions exposed at /apis
    // 获取/apis下面的所有Group
	apiGroupList = &metav1.APIGroupList{}
	err = d.restClient.Get().AbsPath("/apis").Do(context.TODO()).Into(apiGroupList)
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}
	// to be compatible with a v1.0 server, if it's a 403 or 404, ignore and return whatever we got from /api
	if err != nil && (errors.IsNotFound(err) || errors.IsForbidden(err)) {
		apiGroupList = &metav1.APIGroupList{}
	}

	// prepend the group retrieved from /api to the list if not empty
	if len(v.Versions) != 0 {
		apiGroupList.Groups = append([]metav1.APIGroup{apiGroup}, apiGroupList.Groups...)
	}
	return apiGroupList, nil
}
```

## 缓存机制
CachedDiscovery客户端是一种带有缓存机制的Discovery客户端。该客户端会缓存资源相关的信息到本地，目录为`.kube/cache`和`.kube/http-cache`以减少对api-server的压力，默认每10分钟更新一次
```go
// vendor/k8s.io/client-go/discovery/cached/disk/cached_discovery.go
func (d *CachedDiscoveryClient) getCachedFile(filename string) ([]byte, error) {
	// after invalidation ignore cache files not created by this process
	d.mutex.Lock()
	_, ourFile := d.ourFiles[filename]
	if d.invalidated && !ourFile {
		d.mutex.Unlock()
		return nil, errors.New("cache invalidated")
	}
	d.mutex.Unlock()

	file, err := os.Open(filename)
	if err != nil {
		return nil, err
	}
	defer file.Close()

	fileInfo, err := file.Stat()
	if err != nil {
		return nil, err
	}

	if time.Now().After(fileInfo.ModTime().Add(d.ttl)) {
		return nil, errors.New("cache expired")
	}

	// the cache is present and its valid.  Try to read and use it.
	cachedBytes, err := ioutil.ReadAll(file)
	if err != nil {
		return nil, err
	}

	d.mutex.Lock()
	defer d.mutex.Unlock()
	d.fresh = d.fresh && ourFile

	return cachedBytes, nil
}
```