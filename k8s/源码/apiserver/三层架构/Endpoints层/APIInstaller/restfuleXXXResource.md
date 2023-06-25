# restfulXXXResource
restfulXXXResource是registerResourceHandlers中调用的，用于获取`restful.RouteFunction`函数的函数

## restful.RouteFunction
restful.RouteFunction是go-restful定义的Route的handler函数
```go
// Request与Response均是go-restful库下的结构体
type RouteFunction func(*Request, *Response)
```

## restfulXXXResource
可以看到restfulXXXResource返回的`restful.RouteFunction`函数调用对应的handlers.XXXResource函数获取http.ServeHTTP函数来处理
```go
// k8s.io/apiserver/pkg/endpoints/installer.go
func restfulListResource(r rest.Lister, rw rest.Watcher, scope handlers.RequestScope, forceWatch bool, minRequestTimeout time.Duration) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.ListResource(r, rw, &scope, forceWatch, minRequestTimeout)(res.ResponseWriter, req.Request)
	}
}

func restfulCreateNamedResource(r rest.NamedCreater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.CreateNamedResource(r, &scope, admit)(res.ResponseWriter, req.Request)
	}
}

func restfulCreateResource(r rest.Creater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.CreateResource(r, &scope, admit)(res.ResponseWriter, req.Request)
	}
}

func restfulDeleteResource(r rest.GracefulDeleter, allowsOptions bool, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.DeleteResource(r, allowsOptions, &scope, admit)(res.ResponseWriter, req.Request)
	}
}

func restfulDeleteCollection(r rest.CollectionDeleter, checkBody bool, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.DeleteCollection(r, checkBody, &scope, admit)(res.ResponseWriter, req.Request)
	}
}

func restfulUpdateResource(r rest.Updater, scope handlers.RequestScope, admit admission.Interface) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.UpdateResource(r, &scope, admit)(res.ResponseWriter, req.Request)
	}
}

func restfulPatchResource(r rest.Patcher, scope handlers.RequestScope, admit admission.Interface, supportedTypes []string) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.PatchResource(r, &scope, admit, supportedTypes)(res.ResponseWriter, req.Request)
	}
}

func restfulGetResource(r rest.Getter, scope handlers.RequestScope) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.GetResource(r, &scope)(res.ResponseWriter, req.Request)
	}
}

func restfulGetResourceWithOptions(r rest.GetterWithOptions, scope handlers.RequestScope, isSubresource bool) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.GetResourceWithOptions(r, &scope, isSubresource)(res.ResponseWriter, req.Request)
	}
}

func restfulConnectResource(connecter rest.Connecter, scope handlers.RequestScope, admit admission.Interface, restPath string, isSubresource bool) restful.RouteFunction {
	return func(req *restful.Request, res *restful.Response) {
		handlers.ConnectResource(connecter, &scope, admit, restPath, isSubresource)(res.ResponseWriter, req.Request)
	}
}
```