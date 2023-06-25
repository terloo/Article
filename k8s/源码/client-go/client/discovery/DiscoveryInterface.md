# DiscoveryInterface
Discovery客户端的接口

## 
```go
// k8s.io/client-go/discovery/discovery_client.go

// 带有缓存的Discovery
type CachedDiscoveryInterface interface {
	DiscoveryInterface
	// Fresh is supposed to tell the caller whether or not to retry if the cache
	// fails to find something (false = retry, true = no need to retry).
	//
	// TODO: this needs to be revisited, this interface can't be locked properly
	// and doesn't make a lot of sense.
	Fresh() bool
	// Invalidate enforces that no cached data that is older than the current time
	// is used.
	Invalidate()
}

type DiscoveryInterface interface {
	RESTClient() restclient.Interface
	ServerGroupsInterface
	ServerResourcesInterface
	ServerVersionInterface
	OpenAPISchemaInterface
}

type ServerGroupsInterface interface {
	// ServerGroups returns the supported groups, with information like supported versions and the
	// preferred version.
	ServerGroups() (*metav1.APIGroupList, error)
}

// ServerResourcesInterface has methods for obtaining supported resources on the API server
type ServerResourcesInterface interface {
	// ServerResourcesForGroupVersion returns the supported resources for a group and version.
	ServerResourcesForGroupVersion(groupVersion string) (*metav1.APIResourceList, error)
	// ServerResources returns the supported resources for all groups and versions.
	//
	// The returned resource list might be non-nil with partial results even in the case of
	// non-nil error.
	//
	// Deprecated: use ServerGroupsAndResources instead.
	ServerResources() ([]*metav1.APIResourceList, error)
	// ServerGroupsAndResources returns the supported groups and resources for all groups and versions.
	//
	// The returned group and resource lists might be non-nil with partial results even in the
	// case of non-nil error.
	ServerGroupsAndResources() ([]*metav1.APIGroup, []*metav1.APIResourceList, error)
	// ServerPreferredResources returns the supported resources with the version preferred by the
	// server.
	//
	// The returned group and resource lists might be non-nil with partial results even in the
	// case of non-nil error.
	ServerPreferredResources() ([]*metav1.APIResourceList, error)
	// ServerPreferredNamespacedResources returns the supported namespaced resources with the
	// version preferred by the server.
	//
	// The returned resource list might be non-nil with partial results even in the case of
	// non-nil error.
	ServerPreferredNamespacedResources() ([]*metav1.APIResourceList, error)
}

// ServerVersionInterface has a method for retrieving the server's version.
type ServerVersionInterface interface {
	// ServerVersion retrieves and parses the server's version (git version).
	ServerVersion() (*version.Info, error)
}

// OpenAPISchemaInterface has a method to retrieve the open API schema.
type OpenAPISchemaInterface interface {
	// OpenAPISchema retrieves and parses the swagger API schema the server supports.
	OpenAPISchema() (*openapi_v2.Document, error)
}
```