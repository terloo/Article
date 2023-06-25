# crdHandler
专门用于处理对CRD资源请求的Handler

## 结构体
```go
// vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_handler.go
type crdHandler struct {
	versionDiscoveryHandler *versionDiscoveryHandler
	groupDiscoveryHandler   *groupDiscoveryHandler

	customStorageLock sync.Mutex
	// 用于存储crdStorageMap
	customStorage atomic.Value

	crdLister listers.CustomResourceDefinitionLister

	delegate          http.Handler
	restOptionsGetter generic.RESTOptionsGetter
	admission         admission.Interface

	establishingController *establish.EstablishingController

	// MasterCount is used to implement sleep to improve
	// CRD establishing process for HA clusters.
	masterCount int

	converterFactory *conversion.CRConverterFactory

	// so that we can do create on update.
	authorizer authorizer.Authorizer

	// request timeout we should delay storage teardown for
	requestTimeout time.Duration

	// minRequestTimeout applies to CR's list/watch calls
	minRequestTimeout time.Duration

	// staticOpenAPISpec is used as a base for the schema of CR's for the
	// purpose of managing fields, it is how CR handlers get the structure
	// of TypeMeta and ObjectMeta
	staticOpenAPISpec *spec.Swagger

	// The limit on the request size that would be accepted and decoded in a write request
	// 0 means no limit.
	maxRequestBodyBytes int64
}

type crdStorageMap map[types.UID]*crdInfo

type crdInfo struct {
	// spec and acceptedNames are used to compare against if a change is made on a CRD. We only update
	// the storage if one of these changes.
	spec          *apiextensionsv1.CustomResourceDefinitionSpec
	acceptedNames *apiextensionsv1.CustomResourceDefinitionNames

	// Deprecated per version
	deprecated map[string]bool

	// Warnings per version
	warnings map[string][]string

	// Storage per version
	storages map[string]customresource.CustomResourceStorage

	// Request scope per version
	requestScopes map[string]*handlers.RequestScope

	// Scale scope per version
	scaleRequestScopes map[string]*handlers.RequestScope

	// Status scope per version
	statusRequestScopes map[string]*handlers.RequestScope

	// storageVersion is the CRD version used when storing the object in etcd.
	storageVersion string

	waitGroup *utilwaitgroup.SafeWaitGroup
}
```

## 构造函数
```go
// vendor/k8s.io/apiextensions-apiserver/apiserver/customresource_handler.go
func NewCustomResourceDefinitionHandler(
	versionDiscoveryHandler *versionDiscoveryHandler,
	groupDiscoveryHandler *groupDiscoveryHandler,
	crdInformer informers.CustomResourceDefinitionInformer,
	delegate http.Handler,
	restOptionsGetter generic.RESTOptionsGetter,
	admission admission.Interface,
	establishingController *establish.EstablishingController,
	serviceResolver webhook.ServiceResolver,
	authResolverWrapper webhook.AuthenticationInfoResolverWrapper,
	masterCount int,
	authorizer authorizer.Authorizer,
	requestTimeout time.Duration,
	minRequestTimeout time.Duration,
	staticOpenAPISpec *spec.Swagger,
	maxRequestBodyBytes int64) (*crdHandler, error) {
	ret := &crdHandler{
		versionDiscoveryHandler: versionDiscoveryHandler,
		groupDiscoveryHandler:   groupDiscoveryHandler,
		customStorage:           atomic.Value{},
		crdLister:               crdInformer.Lister(),
		delegate:                delegate,
		restOptionsGetter:       restOptionsGetter,
		admission:               admission,
		establishingController:  establishingController,
		masterCount:             masterCount,
		authorizer:              authorizer,
		requestTimeout:          requestTimeout,
		minRequestTimeout:       minRequestTimeout,
		staticOpenAPISpec:       staticOpenAPISpec,
		maxRequestBodyBytes:     maxRequestBodyBytes,
	}
	crdInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    ret.createCustomResourceDefinition,
		UpdateFunc: ret.updateCustomResourceDefinition,
		DeleteFunc: func(obj interface{}) {
			ret.removeDeadStorage()
		},
	})
	crConverterFactory, err := conversion.NewCRConverterFactory(serviceResolver, authResolverWrapper)
	if err != nil {
		return nil, err
	}
	ret.converterFactory = crConverterFactory

	ret.customStorage.Store(crdStorageMap{})

	return ret, nil
}
```

## ServeHTTP
```go
// vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/customresource_handler.go
func (r *crdHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	ctx := req.Context()
	requestInfo, ok := apirequest.RequestInfoFrom(ctx)
	if !ok {
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("no RequestInfo found in the context")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}
	if !requestInfo.IsResourceRequest {
		pathParts := splitPath(requestInfo.Path)
		// only match /apis/<group>/<version>
		// only registered under /apis
		// 如果是discovery请求，交由discovery处理
		if len(pathParts) == 3 {
			r.versionDiscoveryHandler.ServeHTTP(w, req)
			return
		}
		// only match /apis/<group>
		if len(pathParts) == 2 {
			r.groupDiscoveryHandler.ServeHTTP(w, req)
			return
		}

		r.delegate.ServeHTTP(w, req)
		return
	}

	// 先检查是否有对应的crd
	crdName := requestInfo.Resource + "." + requestInfo.APIGroup
	crd, err := r.crdLister.Get(crdName)
	if apierrors.IsNotFound(err) {
		r.delegate.ServeHTTP(w, req)
		return
	}
	if err != nil {
		utilruntime.HandleError(err)
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("error resolving resource")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}

	// if the scope in the CRD and the scope in request differ (with exception of the verbs in possiblyAcrossAllNamespacesVerbs
	// for namespaced resources), pass request to the delegate, which is supposed to lead to a 404.
	// 检查资源是否为命名空间
	namespacedCRD, namespacedReq := crd.Spec.Scope == apiextensionsv1.NamespaceScoped, len(requestInfo.Namespace) > 0
	if !namespacedCRD && namespacedReq {
		r.delegate.ServeHTTP(w, req)
		return
	}
	if namespacedCRD && !namespacedReq && !possiblyAcrossAllNamespacesVerbs.Has(requestInfo.Verb) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	// 检查是否有对应版本
	if !apiextensionshelpers.HasServedCRDVersion(crd, requestInfo.APIVersion) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	// There is a small chance that a CRD is being served because NamesAccepted condition is true,
	// but it becomes "unserved" because another names update leads to a conflict
	// and EstablishingController wasn't fast enough to put the CRD into the Established condition.
	// We accept this as the problem is small and self-healing.
	// 检查CRD的状态
	if !apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.NamesAccepted) &&
		!apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Established) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	terminating := apiextensionshelpers.IsCRDConditionTrue(crd, apiextensionsv1.Terminating)

	crdInfo, err := r.getOrCreateServingInfoFor(crd.UID, crd.Name)
	if apierrors.IsNotFound(err) {
		r.delegate.ServeHTTP(w, req)
		return
	}
	if err != nil {
		utilruntime.HandleError(err)
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("error resolving resource")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}
	if !hasServedCRDVersion(crdInfo.spec, requestInfo.APIVersion) {
		r.delegate.ServeHTTP(w, req)
		return
	}

	deprecated := crdInfo.deprecated[requestInfo.APIVersion]
	for _, w := range crdInfo.warnings[requestInfo.APIVersion] {
		warning.AddWarning(req.Context(), "", w)
	}

	verb := strings.ToUpper(requestInfo.Verb)
	resource := requestInfo.Resource
	subresource := requestInfo.Subresource
	scope := metrics.CleanScope(requestInfo)
	supportedTypes := []string{
		string(types.JSONPatchType),
		string(types.MergePatchType),
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.ServerSideApply) {
		supportedTypes = append(supportedTypes, string(types.ApplyPatchType))
	}

	var handlerFunc http.HandlerFunc
	subresources, err := apiextensionshelpers.GetSubresourcesForVersion(crd, requestInfo.APIVersion)
	if err != nil {
		utilruntime.HandleError(err)
		responsewriters.ErrorNegotiated(
			apierrors.NewInternalError(fmt.Errorf("could not properly serve the subresource")),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
		return
	}
	switch {
	case subresource == "status" && subresources != nil && subresources.Status != nil:
		handlerFunc = r.serveStatus(w, req, requestInfo, crdInfo, terminating, supportedTypes)
	case subresource == "scale" && subresources != nil && subresources.Scale != nil:
		handlerFunc = r.serveScale(w, req, requestInfo, crdInfo, terminating, supportedTypes)
	case len(subresource) == 0:
		handlerFunc = r.serveResource(w, req, requestInfo, crdInfo, crd, terminating, supportedTypes)
	default:
		responsewriters.ErrorNegotiated(
			apierrors.NewNotFound(schema.GroupResource{Group: requestInfo.APIGroup, Resource: requestInfo.Resource}, requestInfo.Name),
			Codecs, schema.GroupVersion{Group: requestInfo.APIGroup, Version: requestInfo.APIVersion}, w, req,
		)
	}

	if handlerFunc != nil {
		handlerFunc = metrics.InstrumentHandlerFunc(verb, requestInfo.APIGroup, requestInfo.APIVersion, resource, subresource, scope, metrics.APIServerComponent, deprecated, "", handlerFunc)
		handler := genericfilters.WithWaitGroup(handlerFunc, longRunningFilter, crdInfo.waitGroup)
		handler.ServeHTTP(w, req)
		return
	}
}
```