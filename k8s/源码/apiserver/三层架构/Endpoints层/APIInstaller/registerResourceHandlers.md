# registerResourceHandlers
registerResourceHandlers是APIInstaller中将资源对象的Registry层注册到go-restful的WebService中使用的方法

## 方法流程
1. 获取该资源的GVK
2. 判断资源是否为子资源
3. 判断资源是否为命名空间级别
4. 判断Registry层支持的操作
5. 实例化xxxOptions类，用于go-rest文档
6. 根据资源对象实现的Registry接口，整合所有Verbs
7. 构建reqScope对象，用于保存handler所使用的上下文
8. 构建go-restful的Route，注册到ws
9.  构建resourceInfo (该特性默认关闭)
10. 构建并填充APIResource用于服务发现

## 获取该资源的GVK
由于Registry层是GR维度的，所以要获取该GR对应GV的GVK
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...
	fqKindToRegister, err := GetResourceKind(a.group.GroupVersion, storage, a.group.Typer)
	if err != nil {
		return nil, nil, err
	}
	// ...
	kind := fqKindToRegister.Kind
	// ...
}

func GetResourceKind(groupVersion schema.GroupVersion, storage rest.Storage, typer runtime.ObjectTyper) (schema.GroupVersionKind, error) {
	// Let the storage tell us exactly what GVK it has
	// 如果Registry层实现了GroupVersionKindProvider，那么直接调用，来获取GVK
	if gvkProvider, ok := storage.(rest.GroupVersionKindProvider); ok {
		return gvkProvider.GroupVersionKind(groupVersion), nil
	}

	// New一个资源对象，该对象是内部版本
	object := storage.New()
	// 通过Scheme获取该结构体所有可能的GVK
	fqKinds, _, err := typer.ObjectKinds(object)
	if err != nil {
		return schema.GroupVersionKind{}, err
	}

	// a given go type can have multiple potential fully qualified kinds.  Find the one that corresponds with the group
	// we're trying to register here
	// 如果在所有可能的GVK中有与传入的GV相同的G，拼上K后返回
	fqKindToRegister := schema.GroupVersionKind{}
	for _, fqKind := range fqKinds {
		if fqKind.Group == groupVersion.Group {
			fqKindToRegister = groupVersion.WithKind(fqKind.Kind)
			break
		}
	}
	if fqKindToRegister.Empty() {
		return schema.GroupVersionKind{}, fmt.Errorf("unable to locate fully qualified kind for %v: found %v when registering for %v", reflect.TypeOf(object), fqKinds, groupVersion)
	}

	// group is guaranteed to match based on the check above
	return fqKindToRegister, nil
}
```

## 判断资源是否为子资源
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	resource, subresource, err := splitSubresource(path)
	if err != nil {
		return nil, nil, err
	}

	// ...
	isSubresource := len(subresource) > 0

	// ...
}

// 返回参数  主资源 子资源 error
func splitSubresource(path string) (string, string, error) {
	var resource, subresource string
	switch parts := strings.Split(path, "/"); len(parts) {
	case 2:
		resource, subresource = parts[0], parts[1]
	case 1:
		resource = parts[0]
	default:
		// TODO: support deeper paths
		return "", "", fmt.Errorf("api_installer allows only one or two segment paths (resource or resource/subresource)")
	}
	return resource, subresource, nil
}
```

## 判断资源是否为命名空间级别
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	// 如果是子资源，则判断其父资源是否为命名空间级别
	var namespaceScoped bool
	if isSubresource {
		parentStorage, ok := a.group.Storage[resource]
		if !ok {
			return nil, nil, fmt.Errorf("missing parent storage: %q", resource)
		}
		scoper, ok := parentStorage.(rest.Scoper)
		if !ok {
			return nil, nil, fmt.Errorf("%q must implement scoper", resource)
		}
		namespaceScoped = scoper.NamespaceScoped()

	} else {
		scoper, ok := storage.(rest.Scoper)
		if !ok {
			return nil, nil, fmt.Errorf("%q must implement scoper", resource)
		}
		namespaceScoped = scoper.NamespaceScoped()
	}

	// ...
}
```

## 判断Registry层支持的操作
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	creater, isCreater := storage.(rest.Creater)
	namedCreater, isNamedCreater := storage.(rest.NamedCreater)
	lister, isLister := storage.(rest.Lister)
	getter, isGetter := storage.(rest.Getter)
	getterWithOptions, isGetterWithOptions := storage.(rest.GetterWithOptions)
	gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
	collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
	updater, isUpdater := storage.(rest.Updater)
	patcher, isPatcher := storage.(rest.Patcher)
	watcher, isWatcher := storage.(rest.Watcher)
	connecter, isConnecter := storage.(rest.Connecter)
	storageMeta, isMetadata := storage.(rest.StorageMetadata)
	storageVersionProvider, isStorageVersionProvider := storage.(rest.StorageVersionProvider)
	gvAcceptor, _ := storage.(rest.GroupVersionAcceptor)
	// 如果没有实现StorageMetadata则使用defaultStorageMetadata，该结构体方法均返回nil
	if !isMetadata {
		storageMeta = defaultStorageMetadata{}
	}

	// 实现了NamedCreater也当作实现了Creater
	if isNamedCreater {
		isCreater = true
	}

	// ...
}
```

## 实例化xxxOptions类，用于go-restful配置参数
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	// 这个GV硬编码为 corev1
	optionsExternalVersion := a.group.GroupVersion
	if a.group.OptionsExternalVersion != nil {
		optionsExternalVersion = *a.group.OptionsExternalVersion
	}

	// ...

	var versionedList interface{}
	if isLister {
		list := lister.NewList()
		listGVKs, _, err := a.group.Typer.ObjectKinds(list)
		if err != nil {
			return nil, nil, err
		}
		versionedListPtr, err := a.group.Creater.New(a.group.GroupVersion.WithKind(listGVKs[0].Kind))
		if err != nil {
			return nil, nil, err
		}
		versionedList = indirectArbitraryPointer(versionedListPtr)
	}

	// 实例化ListOptions，CreateOptions，PatchOptions，UpdateOptions，DeleteOptions
	versionedListOptions, err := a.group.Creater.New(optionsExternalVersion.WithKind("ListOptions"))
	if err != nil {
		return nil, nil, err
	}
	versionedCreateOptions, err := a.group.Creater.New(optionsExternalVersion.WithKind("CreateOptions"))
	if err != nil {
		return nil, nil, err
	}
	versionedPatchOptions, err := a.group.Creater.New(optionsExternalVersion.WithKind("PatchOptions"))
	if err != nil {
		return nil, nil, err
	}
	versionedUpdateOptions, err := a.group.Creater.New(optionsExternalVersion.WithKind("UpdateOptions"))
	if err != nil {
		return nil, nil, err
	}

	var versionedDeleteOptions runtime.Object
	var versionedDeleterObject interface{}
	deleteReturnsDeletedObject := false
	if isGracefulDeleter {
		versionedDeleteOptions, err = a.group.Creater.New(optionsExternalVersion.WithKind("DeleteOptions"))
		if err != nil {
			return nil, nil, err
		}
		versionedDeleterObject = indirectArbitraryPointer(versionedDeleteOptions)

		if mayReturnFullObjectDeleter, ok := storage.(rest.MayReturnFullObjectDeleter); ok {
			deleteReturnsDeletedObject = mayReturnFullObjectDeleter.DeleteReturnsDeletedObject()
		}
	}


	// 构建Status对象
	versionedStatusPtr, err := a.group.Creater.New(optionsExternalVersion.WithKind("Status"))
	if err != nil {
		return nil, nil, err
	}
	versionedStatus := indirectArbitraryPointer(versionedStatusPtr)

	// 构建GetOptions对象，目前只有PodLog实现此接口
	var (
		getOptions             runtime.Object
		versionedGetOptions    runtime.Object
		getOptionsInternalKind schema.GroupVersionKind
		getSubpath             bool
	)
	if isGetterWithOptions {
		getOptions, getSubpath, _ = getterWithOptions.NewGetOptions()
		getOptionsInternalKinds, _, err := a.group.Typer.ObjectKinds(getOptions)
		if err != nil {
			return nil, nil, err
		}
		getOptionsInternalKind = getOptionsInternalKinds[0]
		versionedGetOptions, err = a.group.Creater.New(a.group.GroupVersion.WithKind(getOptionsInternalKind.Kind))
		if err != nil {
			versionedGetOptions, err = a.group.Creater.New(optionsExternalVersion.WithKind(getOptionsInternalKind.Kind))
			if err != nil {
				return nil, nil, err
			}
		}
		isGetter = true
	}

	// 构建WatchEvent对象
	var versionedWatchEvent interface{}
	if isWatcher {
		versionedWatchEventPtr, err := a.group.Creater.New(a.group.GroupVersion.WithKind("WatchEvent"))
		if err != nil {
			return nil, nil, err
		}
		versionedWatchEvent = indirectArbitraryPointer(versionedWatchEventPtr)
	}

	// 构建ConnectOptions对象，目前只有几个实现了代理的对象实现此接口
	var (
		connectOptions             runtime.Object
		versionedConnectOptions    runtime.Object
		connectOptionsInternalKind schema.GroupVersionKind
		connectSubpath             bool
	)
	if isConnecter {
		connectOptions, connectSubpath, _ = connecter.NewConnectOptions()
		if connectOptions != nil {
			connectOptionsInternalKinds, _, err := a.group.Typer.ObjectKinds(connectOptions)
			if err != nil {
				return nil, nil, err
			}

			connectOptionsInternalKind = connectOptionsInternalKinds[0]
			versionedConnectOptions, err = a.group.Creater.New(a.group.GroupVersion.WithKind(connectOptionsInternalKind.Kind))
			if err != nil {
				versionedConnectOptions, err = a.group.Creater.New(optionsExternalVersion.WithKind(connectOptionsInternalKind.Kind))
				if err != nil {
					return nil, nil, err
				}
			}
		}
	}

	// 资源对象是否可以WatchList
	allowWatchList := isWatcher && isLister // watching on lists is allowed only for kinds that support both watch and list.
	nameParam := ws.PathParameter("name", "name of the "+kind).DataType("string")
	pathParam := ws.PathParameter("path", "path to the resource").DataType("string")

	params := []*restful.Parameter{}
	actions := []action{}

	// ...

	// 资源对象是否可以以Table的格式格式化，Registry层实现
	tableProvider, isTableProvider := storage.(rest.TableConvertor)
	if isLister && !isTableProvider {
		// All listers must implement TableProvider
		return nil, nil, fmt.Errorf("%q must implement TableConvertor", resource)
	}

	// ...
}
```

## 根据资源对象实现的Registry接口，整合所有Verbs
顺带给APIResource服务发现对象赋值Name NameSpaced Kind字段
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	// 分别处理命名空间级和非命名空间级别
	switch {
		case !namespaceScoped:
			// Handle non-namespace scoped resources like nodes.
			resourcePath := resource
			resourceParams := params
			itemPath := resourcePath + "/{name}"
			nameParams := append(params, nameParam)
			proxyParams := append(nameParams, pathParam)
			suffix := ""
			if isSubresource {
				suffix = "/" + subresource
				itemPath = itemPath + suffix
				resourcePath = itemPath
				resourceParams = nameParams
			}
			apiResource.Name = path
			apiResource.Namespaced = false
			apiResource.Kind = resourceKind
			namer := handlers.ContextBasedNaming{
				SelfLinker:         a.group.Linker,
				ClusterScoped:      true,
				SelfLinkPathPrefix: gpath.Join(a.prefix, resource) + "/",
				SelfLinkPathSuffix: suffix,
			}

			// Handler for standard REST verbs (GET, PUT, POST and DELETE).
			// Add actions at the resource path: /api/apiVersion/resource
			actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
			actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
			actions = appendIf(actions, action{"DELETECOLLECTION", resourcePath, resourceParams, namer, false}, isCollectionDeleter)
			// DEPRECATED in 1.11
			actions = appendIf(actions, action{"WATCHLIST", "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)

			// Add actions at the item path: /api/apiVersion/resource/{name}
			actions = appendIf(actions, action{"GET", itemPath, nameParams, namer, false}, isGetter)
			if getSubpath {
				actions = appendIf(actions, action{"GET", itemPath + "/{path:*}", proxyParams, namer, false}, isGetter)
			}
			actions = appendIf(actions, action{"PUT", itemPath, nameParams, namer, false}, isUpdater)
			actions = appendIf(actions, action{"PATCH", itemPath, nameParams, namer, false}, isPatcher)
			actions = appendIf(actions, action{"DELETE", itemPath, nameParams, namer, false}, isGracefulDeleter)
			// DEPRECATED in 1.11
			actions = appendIf(actions, action{"WATCH", "watch/" + itemPath, nameParams, namer, false}, isWatcher)
			actions = appendIf(actions, action{"CONNECT", itemPath, nameParams, namer, false}, isConnecter)
			actions = appendIf(actions, action{"CONNECT", itemPath + "/{path:*}", proxyParams, namer, false}, isConnecter && connectSubpath)
		default:
			namespaceParamName := "namespaces"
			// Handler for standard REST verbs (GET, PUT, POST and DELETE).
			namespaceParam := ws.PathParameter("namespace", "object name and auth scope, such as for teams and projects").DataType("string")
			namespacedPath := namespaceParamName + "/{namespace}/" + resource
			namespaceParams := []*restful.Parameter{namespaceParam}

			resourcePath := namespacedPath
			resourceParams := namespaceParams
			itemPath := namespacedPath + "/{name}"
			nameParams := append(namespaceParams, nameParam)
			proxyParams := append(nameParams, pathParam)
			itemPathSuffix := ""
			if isSubresource {
				itemPathSuffix = "/" + subresource
				itemPath = itemPath + itemPathSuffix
				resourcePath = itemPath
				resourceParams = nameParams
			}
			apiResource.Name = path
			apiResource.Namespaced = true
			apiResource.Kind = resourceKind
			namer := handlers.ContextBasedNaming{
				SelfLinker:         a.group.Linker,
				ClusterScoped:      false,
				SelfLinkPathPrefix: gpath.Join(a.prefix, namespaceParamName) + "/",
				SelfLinkPathSuffix: itemPathSuffix,
			}

			actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
			actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
			actions = appendIf(actions, action{"DELETECOLLECTION", resourcePath, resourceParams, namer, false}, isCollectionDeleter)
			// DEPRECATED in 1.11
			actions = appendIf(actions, action{"WATCHLIST", "watch/" + resourcePath, resourceParams, namer, false}, allowWatchList)

			actions = appendIf(actions, action{"GET", itemPath, nameParams, namer, false}, isGetter)
			if getSubpath {
				actions = appendIf(actions, action{"GET", itemPath + "/{path:*}", proxyParams, namer, false}, isGetter)
			}
			actions = appendIf(actions, action{"PUT", itemPath, nameParams, namer, false}, isUpdater)
			actions = appendIf(actions, action{"PATCH", itemPath, nameParams, namer, false}, isPatcher)
			actions = appendIf(actions, action{"DELETE", itemPath, nameParams, namer, false}, isGracefulDeleter)
			// DEPRECATED in 1.11
			actions = appendIf(actions, action{"WATCH", "watch/" + itemPath, nameParams, namer, false}, isWatcher)
			actions = appendIf(actions, action{"CONNECT", itemPath, nameParams, namer, false}, isConnecter)
			actions = appendIf(actions, action{"CONNECT", itemPath + "/{path:*}", proxyParams, namer, false}, isConnecter && connectSubpath)

			// list or post across namespace.
			// For ex: LIST all pods in all namespaces by sending a LIST request at /api/apiVersion/pods.
			// TODO: more strongly type whether a resource allows these actions on "all namespaces" (bulk delete)
			if !isSubresource {
				actions = appendIf(actions, action{"LIST", resource, params, namer, true}, isLister)
				// DEPRECATED in 1.11
				actions = appendIf(actions, action{"WATCHLIST", "watch/" + resource, params, namer, true}, allowWatchList)
			}
		}

	// ...
}
```

## 构建reqScope对象，用于保存handler所使用的上下文
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
// reqScope对象，用于保存handler所使用的上下文
	reqScope := handlers.RequestScope{
		Serializer:      a.group.Serializer,
		ParameterCodec:  a.group.ParameterCodec,
		Creater:         a.group.Creater,
		Convertor:       a.group.Convertor,
		Defaulter:       a.group.Defaulter,
		Typer:           a.group.Typer,
		UnsafeConvertor: a.group.UnsafeConvertor,
		Authorizer:      a.group.Authorizer,

		EquivalentResourceMapper: a.group.EquivalentResourceRegistry,

		// TODO: Check for the interface on storage
		TableConvertor: tableProvider,

		// TODO: This seems wrong for cross-group subresources. It makes an assumption that a subresource and its parent are in the same group version. Revisit this.
		Resource:    a.group.GroupVersion.WithResource(resource),
		Subresource: subresource,
		Kind:        fqKindToRegister,

		AcceptsGroupVersionDelegate: gvAcceptor,

		HubGroupVersion: schema.GroupVersion{Group: fqKindToRegister.Group, Version: runtime.APIVersionInternal},

		MetaGroupVersion: metav1.SchemeGroupVersion,

		MaxRequestBodyBytes: a.group.MaxRequestBodyBytes,
	}
	if a.group.MetaGroupVersion != nil {
		reqScope.MetaGroupVersion = *a.group.MetaGroupVersion
	}
	if a.group.OpenAPIModels != nil && utilfeature.DefaultFeatureGate.Enabled(features.ServerSideApply) {
		reqScope.FieldManager, err = fieldmanager.NewDefaultFieldManager(
			a.group.TypeConverter,
			a.group.UnsafeConvertor,
			a.group.Defaulter,
			a.group.Creater,
			fqKindToRegister,
			reqScope.HubGroupVersion,
			subresource,
			resetFields,
		)
		if err != nil {
			return nil, nil, fmt.Errorf("failed to create field manager: %v", err)
		}
	}
```

## 构建go-restful的Route，注册到ws
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	// 先配置ws可以产生响应类型
	for _, s := range a.group.Serializer.SupportedMediaTypes() {
		if len(s.MediaTypeSubType) == 0 || len(s.MediaTypeType) == 0 {
			return nil, nil, fmt.Errorf("all serializers in the group Serializer must have MediaTypeType and MediaTypeSubType set: %s", s.MediaType)
		}
	}
	mediaTypes, streamMediaTypes := negotiation.MediaTypesForSerializer(a.group.Serializer)
	allMediaTypes := append(mediaTypes, streamMediaTypes...)
	ws.Produces(allMediaTypes...)

	// ...

	kubeVerbs := map[string]struct{}{}
	for _, action := range actions {
		producedObject := storageMeta.ProducesObject(action.Verb)
		if producedObject == nil {
			producedObject = defaultVersionedObject
		}
		reqScope.Namer = action.Namer

		requestScope := "cluster"
		var namespaced string
		var operationSuffix string
		if apiResource.Namespaced {
			requestScope = "namespace"
			namespaced = "Namespaced"
		}
		if strings.HasSuffix(action.Path, "/{path:*}") {
			requestScope = "resource"
			operationSuffix = operationSuffix + "WithPath"
		}
		if strings.Index(action.Path, "/{name}") != -1 || action.Verb == "POST" {
			requestScope = "resource"
		}
		if action.AllNamespaces {
			requestScope = "cluster"
			operationSuffix = operationSuffix + "ForAllNamespaces"
			namespaced = ""
		}

		if kubeVerb, found := toDiscoveryKubeVerb[action.Verb]; found {
			if len(kubeVerb) != 0 {
				kubeVerbs[kubeVerb] = struct{}{}
			}
		} else {
			return nil, nil, fmt.Errorf("unknown action verb for discovery: %s", action.Verb)
		}

		// 每个资源的每个请求都是一个Route
		routes := []*restful.RouteBuilder{}

		// If there is a subresource, kind should be the parent's kind.
		if isSubresource {
			parentStorage, ok := a.group.Storage[resource]
			if !ok {
				return nil, nil, fmt.Errorf("missing parent storage: %q", resource)
			}

			fqParentKind, err := GetResourceKind(a.group.GroupVersion, parentStorage, a.group.Typer)
			if err != nil {
				return nil, nil, err
			}
			kind = fqParentKind.Kind
		}

		verbOverrider, needOverride := storage.(StorageMetricsOverride)

		// accumulate endpoint-level warnings
		var (
			warnings       []string
			deprecated     bool
			removedRelease string
		)

		{
			versionedPtrWithGVK := versionedPtr.DeepCopyObject()
			versionedPtrWithGVK.GetObjectKind().SetGroupVersionKind(fqKindToRegister)
			currentMajor, currentMinor, _ := deprecation.MajorMinor(versioninfo.Get())
			deprecated = deprecation.IsDeprecated(versionedPtrWithGVK, currentMajor, currentMinor)
			if deprecated {
				removedRelease = deprecation.RemovedRelease(versionedPtrWithGVK)
				warnings = append(warnings, deprecation.WarningMessage(versionedPtrWithGVK))
			}
		}

		// 构建并包装handler
		switch action.Verb {
		case "GET": // Get a resource.
			var handler restful.RouteFunction
			if isGetterWithOptions {
				handler = restfulGetResourceWithOptions(getterWithOptions, reqScope, isSubresource)
			} else {
				handler = restfulGetResource(getter, reqScope)
			}

			if needOverride {
				// need change the reported verb
				handler = metrics.InstrumentRouteFunc(verbOverrider.OverrideMetricsVerb(action.Verb), group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
			} else {
				handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
			}
			handler = utilwarning.AddWarningsHandler(handler, warnings)

			doc := "read the specified " + kind
			if isSubresource {
				doc = "read " + subresource + " of the specified " + kind
			}
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("read"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				Writes(producedObject)
			if isGetterWithOptions {
				if err := AddObjectParams(ws, route, versionedGetOptions); err != nil {
					return nil, nil, err
				}
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "LIST": // List all resources of a kind.
			doc := "list objects of kind " + kind
			if isSubresource {
				doc = "list " + subresource + " of objects of kind " + kind
			}
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulListResource(lister, watcher, reqScope, false, a.minRequestTimeout))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("list"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), allMediaTypes...)...).
				Returns(http.StatusOK, "OK", versionedList).
				Writes(versionedList)
			if err := AddObjectParams(ws, route, versionedListOptions); err != nil {
				return nil, nil, err
			}
			switch {
			case isLister && isWatcher:
				doc := "list or watch objects of kind " + kind
				if isSubresource {
					doc = "list or watch " + subresource + " of objects of kind " + kind
				}
				route.Doc(doc)
			case isWatcher:
				doc := "watch objects of kind " + kind
				if isSubresource {
					doc = "watch " + subresource + "of objects of kind " + kind
				}
				route.Doc(doc)
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "PUT": // Update a resource.
			doc := "replace the specified " + kind
			if isSubresource {
				doc = "replace " + subresource + " of the specified " + kind
			}
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulUpdateResource(updater, reqScope, admit))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.PUT(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("replace"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				// TODO: in some cases, the API may return a v1.Status instead of the versioned object
				// but currently go-restful can't handle multiple different objects being returned.
				Returns(http.StatusCreated, "Created", producedObject).
				Reads(defaultVersionedObject).
				Writes(producedObject)
			if err := AddObjectParams(ws, route, versionedUpdateOptions, disabledParams...); err != nil {
				return nil, nil, err
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "PATCH": // Partially update a resource
			doc := "partially update the specified " + kind
			if isSubresource {
				doc = "partially update " + subresource + " of the specified " + kind
			}
			supportedTypes := []string{
				string(types.JSONPatchType),
				string(types.MergePatchType),
				string(types.StrategicMergePatchType),
			}
			if utilfeature.DefaultFeatureGate.Enabled(features.ServerSideApply) {
				supportedTypes = append(supportedTypes, string(types.ApplyPatchType))
			}
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulPatchResource(patcher, reqScope, admit, supportedTypes))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.PATCH(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Consumes(supportedTypes...).
				Operation("patch"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				// Patch can return 201 when a server side apply is requested
				Returns(http.StatusCreated, "Created", producedObject).
				Reads(metav1.Patch{}).
				Writes(producedObject)
			if err := AddObjectParams(ws, route, versionedPatchOptions, disabledParams...); err != nil {
				return nil, nil, err
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "POST": // Create a resource.
			var handler restful.RouteFunction
			if isNamedCreater {
				handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
			} else {
				handler = restfulCreateResource(creater, reqScope, admit)
			}
			handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			article := GetArticleForNoun(kind, " ")
			doc := "create" + article + kind
			if isSubresource {
				doc = "create " + subresource + " of" + article + kind
			}
			route := ws.POST(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("create"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				// TODO: in some cases, the API may return a v1.Status instead of the versioned object
				// but currently go-restful can't handle multiple different objects being returned.
				Returns(http.StatusCreated, "Created", producedObject).
				Returns(http.StatusAccepted, "Accepted", producedObject).
				Reads(defaultVersionedObject).
				Writes(producedObject)
			if err := AddObjectParams(ws, route, versionedCreateOptions, disabledParams...); err != nil {
				return nil, nil, err
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "DELETE": // Delete a resource.
			article := GetArticleForNoun(kind, " ")
			doc := "delete" + article + kind
			if isSubresource {
				doc = "delete " + subresource + " of" + article + kind
			}
			deleteReturnType := versionedStatus
			if deleteReturnsDeletedObject {
				deleteReturnType = producedObject
			}
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulDeleteResource(gracefulDeleter, isGracefulDeleter, reqScope, admit))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.DELETE(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("delete"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Writes(deleteReturnType).
				Returns(http.StatusOK, "OK", deleteReturnType).
				Returns(http.StatusAccepted, "Accepted", deleteReturnType)
			if isGracefulDeleter {
				route.Reads(versionedDeleterObject)
				route.ParameterNamed("body").Required(false)
				if err := AddObjectParams(ws, route, versionedDeleteOptions); err != nil {
					return nil, nil, err
				}
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "DELETECOLLECTION":
			doc := "delete collection of " + kind
			if isSubresource {
				doc = "delete collection of " + subresource + " of a " + kind
			}
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulDeleteCollection(collectionDeleter, isCollectionDeleter, reqScope, admit))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.DELETE(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("deletecollection"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Writes(versionedStatus).
				Returns(http.StatusOK, "OK", versionedStatus)
			if isCollectionDeleter {
				route.Reads(versionedDeleterObject)
				route.ParameterNamed("body").Required(false)
				if err := AddObjectParams(ws, route, versionedDeleteOptions); err != nil {
					return nil, nil, err
				}
			}
			if err := AddObjectParams(ws, route, versionedListOptions, "watch", "allowWatchBookmarks"); err != nil {
				return nil, nil, err
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		// deprecated in 1.11
		case "WATCH": // Watch a resource.
			doc := "watch changes to an object of kind " + kind
			if isSubresource {
				doc = "watch changes to " + subresource + " of an object of kind " + kind
			}
			doc += ". deprecated: use the 'watch' parameter with a list operation instead, filtered to a single item with the 'fieldSelector' parameter."
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulListResource(lister, watcher, reqScope, true, a.minRequestTimeout))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("watch"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(allMediaTypes...).
				Returns(http.StatusOK, "OK", versionedWatchEvent).
				Writes(versionedWatchEvent)
			if err := AddObjectParams(ws, route, versionedListOptions); err != nil {
				return nil, nil, err
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		// deprecated in 1.11
		case "WATCHLIST": // Watch all resources of a kind.
			doc := "watch individual changes to a list of " + kind
			if isSubresource {
				doc = "watch individual changes to a list of " + subresource + " of " + kind
			}
			doc += ". deprecated: use the 'watch' parameter with a list operation instead."
			handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulListResource(lister, watcher, reqScope, true, a.minRequestTimeout))
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			route := ws.GET(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed.")).
				Operation("watch"+namespaced+kind+strings.Title(subresource)+"List"+operationSuffix).
				Produces(allMediaTypes...).
				Returns(http.StatusOK, "OK", versionedWatchEvent).
				Writes(versionedWatchEvent)
			if err := AddObjectParams(ws, route, versionedListOptions); err != nil {
				return nil, nil, err
			}
			addParams(route, action.Params)
			routes = append(routes, route)
		case "CONNECT":
			for _, method := range connecter.ConnectMethods() {
				connectProducedObject := storageMeta.ProducesObject(method)
				if connectProducedObject == nil {
					connectProducedObject = "string"
				}
				doc := "connect " + method + " requests to " + kind
				if isSubresource {
					doc = "connect " + method + " requests to " + subresource + " of " + kind
				}
				handler := metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, restfulConnectResource(connecter, reqScope, admit, path, isSubresource))
				handler = utilwarning.AddWarningsHandler(handler, warnings)
				route := ws.Method(method).Path(action.Path).
					To(handler).
					Doc(doc).
					Operation("connect" + strings.Title(strings.ToLower(method)) + namespaced + kind + strings.Title(subresource) + operationSuffix).
					Produces("*/*").
					Consumes("*/*").
					Writes(connectProducedObject)
				if versionedConnectOptions != nil {
					if err := AddObjectParams(ws, route, versionedConnectOptions); err != nil {
						return nil, nil, err
					}
				}
				addParams(route, action.Params)
				routes = append(routes, route)

				// transform ConnectMethods to kube verbs
				if kubeVerb, found := toDiscoveryKubeVerb[method]; found {
					if len(kubeVerb) != 0 {
						kubeVerbs[kubeVerb] = struct{}{}
					}
				}
			}
		default:
			return nil, nil, fmt.Errorf("unrecognized action verb: %s", action.Verb)
		}
		for _, route := range routes {
			// 添加Route到ws，并添加响应头
			route.Metadata(ROUTE_META_GVK, metav1.GroupVersionKind{
				Group:   reqScope.Kind.Group,
				Version: reqScope.Kind.Version,
				Kind:    reqScope.Kind.Kind,
			})
			route.Metadata(ROUTE_META_ACTION, strings.ToLower(action.Verb))
			ws.Route(route)
		}
		// Note: update GetAuthorizerAttributes() when adding a custom handler.
	}
	// ...
}
```

## 构建resourceInfo
**该特性默认关闭**
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...
	var resourceInfo *storageversion.ResourceInfo
	if utilfeature.DefaultFeatureGate.Enabled(features.StorageVersionAPI) &&
		utilfeature.DefaultFeatureGate.Enabled(features.APIServerIdentity) &&
		isStorageVersionProvider &&
		storageVersionProvider.StorageVersion() != nil {

		versioner := storageVersionProvider.StorageVersion()
		encodingGVK, err := getStorageVersionKind(versioner, storage, a.group.Typer)
		if err != nil {
			return nil, nil, err
		}
		decodableVersions := []schema.GroupVersion{}
		if a.group.ConvertabilityChecker != nil {
			decodableVersions = a.group.ConvertabilityChecker.VersionsForGroupKind(fqKindToRegister.GroupKind())
		}
		resourceInfo = &storageversion.ResourceInfo{
			GroupResource: schema.GroupResource{
				Group:    a.group.GroupVersion.Group,
				Resource: apiResource.Name,
			},
			EncodingVersion: encodingGVK.GroupVersion().String(),
			// We record EquivalentResourceMapper first instead of calculate
			// DecodableVersions immediately because API installation must
			// be completed first for us to know equivalent APIs
			EquivalentResourceMapper: a.group.EquivalentResourceRegistry,

			DirectlyDecodableVersions: decodableVersions,
		}
	}
	// ...
}
```

## 构建并填充APIResource用于服务发现
```go
// vendor/k8s.io/apiserver/pkg/endpoints/installer.go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// ...

	var apiResource metav1.APIResource
	// 该特性默认关闭
	if utilfeature.DefaultFeatureGate.Enabled(features.StorageVersionHash) &&
		isStorageVersionProvider &&
		storageVersionProvider.StorageVersion() != nil {
		versioner := storageVersionProvider.StorageVersion()
		gvk, err := getStorageVersionKind(versioner, storage, a.group.Typer)
		if err != nil {
			return nil, nil, err
		}
		apiResource.StorageVersionHash = discovery.StorageVersionHash(gvk.Group, gvk.Version, gvk.Kind)
	}

	// ...

	apiResource.Verbs = make([]string, 0, len(kubeVerbs))
	for kubeVerb := range kubeVerbs {
		apiResource.Verbs = append(apiResource.Verbs, kubeVerb)
	}
	sort.Strings(apiResource.Verbs)

	if shortNamesProvider, ok := storage.(rest.ShortNamesProvider); ok {
		apiResource.ShortNames = shortNamesProvider.ShortNames()
	}
	if categoriesProvider, ok := storage.(rest.CategoriesProvider); ok {
		apiResource.Categories = categoriesProvider.Categories()
	}
	if gvkProvider, ok := storage.(rest.GroupVersionKindProvider); ok {
		gvk := gvkProvider.GroupVersionKind(a.group.GroupVersion)
		apiResource.Group = gvk.Group
		apiResource.Version = gvk.Version
		apiResource.Kind = gvk.Kind
	}

	// ...
}
```
