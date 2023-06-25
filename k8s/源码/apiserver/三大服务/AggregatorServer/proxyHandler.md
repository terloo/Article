# proxyHandler
proxyHandler用于处理APIService的扩展机制，将APIService中指定了代理服务的请求代理到指定的代理服务器，未指定的交由KubeAPIServer进行处理

## 结构体
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/handler_proxy.go
type proxyHandler struct {
	// localDelegate is used to satisfy local APIServices
    // 本地代理，用于没有指定Service字段的APIService，直接代理到kubeAPIServer进行处理
	localDelegate http.Handler

	// proxyCurrentCertKeyContent holds the client cert used to identify this proxy. Backing APIServices use this to confirm the proxy's identity
    // 与代理服务器相通信的配置
	proxyCurrentCertKeyContent certKeyFunc
	proxyTransport             *http.Transport

	// Endpoints based routing to map from cluster IP to routable IP
	serviceResolver ServiceResolver

    // 原子存储，存储handlingInfo
	handlingInfo atomic.Value

	// egressSelector selects the proper egress dialer to communicate with the custom apiserver
	// overwrites proxyTransport dialer if not nil
	egressSelector *egressselector.EgressSelector
}
```

## updateAPIService
传入一个APIService，将其解析为proxyHandlingInfo，存储到handlingInfo字段。
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/handler_proxy.go
func (r *proxyHandler) updateAPIService(apiService *apiregistrationv1api.APIService) {
    // 如果Service字段是nil，说明是本地代理无需解析
	if apiService.Spec.Service == nil {
		r.handlingInfo.Store(proxyHandlingInfo{local: true})
		return
	}

	proxyClientCert, proxyClientKey := r.proxyCurrentCertKeyContent()

	clientConfig := &restclient.Config{
		TLSClientConfig: restclient.TLSClientConfig{
			Insecure:   apiService.Spec.InsecureSkipTLSVerify,
			ServerName: apiService.Spec.Service.Name + "." + apiService.Spec.Service.Namespace + ".svc",
			CertData:   proxyClientCert,
			KeyData:    proxyClientKey,
			CAData:     apiService.Spec.CABundle,
		},
	}
	clientConfig.Wrap(x509metrics.NewMissingSANRoundTripperWrapperConstructor(x509MissingSANCounter))

	newInfo := proxyHandlingInfo{
		name:             apiService.Name,
		restConfig:       clientConfig,
		serviceName:      apiService.Spec.Service.Name,
		serviceNamespace: apiService.Spec.Service.Namespace,
		servicePort:      *apiService.Spec.Service.Port,
		serviceAvailable: apiregistrationv1apihelper.IsAPIServiceConditionTrue(apiService, apiregistrationv1api.Available),
	}
	if r.egressSelector != nil {
		networkContext := egressselector.Cluster.AsNetworkContext()
		var egressDialer utilnet.DialFunc
		egressDialer, err := r.egressSelector.Lookup(networkContext)
		if err != nil {
			klog.Warning(err.Error())
		} else {
			newInfo.restConfig.Dial = egressDialer
		}
	} else if r.proxyTransport != nil && r.proxyTransport.DialContext != nil {
		newInfo.restConfig.Dial = r.proxyTransport.DialContext
	}
	newInfo.proxyRoundTripper, newInfo.transportBuildingError = restclient.TransportFor(newInfo.restConfig)
	if newInfo.transportBuildingError != nil {
		klog.Warning(newInfo.transportBuildingError.Error())
	}
    // 将proxyHandlingInfo存储到handlingInfo字段
	r.handlingInfo.Store(newInfo)
}
```

## ServeHTTP
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/handler_proxy.go
func (r *proxyHandler) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // 如果handlingInfo中没有存值，或者存的handlingInfo.local为true，交由KubeAPIServer进行处理
	value := r.handlingInfo.Load()
	if value == nil {
		r.localDelegate.ServeHTTP(w, req)
		return
	}
	handlingInfo := value.(proxyHandlingInfo)
	if handlingInfo.local {
		if r.localDelegate == nil {
			http.Error(w, "", http.StatusNotFound)
			return
		}
		r.localDelegate.ServeHTTP(w, req)
		return
	}

	if !handlingInfo.serviceAvailable {
		proxyError(w, req, "service unavailable", http.StatusServiceUnavailable)
		return
	}

	if handlingInfo.transportBuildingError != nil {
		proxyError(w, req, handlingInfo.transportBuildingError.Error(), http.StatusInternalServerError)
		return
	}

	user, ok := genericapirequest.UserFrom(req.Context())
	if !ok {
		proxyError(w, req, "missing user", http.StatusInternalServerError)
		return
	}

	// write a new location based on the existing request pointed at the target service
	location := &url.URL{}
	location.Scheme = "https"
	rloc, err := r.serviceResolver.ResolveEndpoint(handlingInfo.serviceNamespace, handlingInfo.serviceName, handlingInfo.servicePort)
	if err != nil {
		klog.Errorf("error resolving %s/%s: %v", handlingInfo.serviceNamespace, handlingInfo.serviceName, err)
		proxyError(w, req, "service unavailable", http.StatusServiceUnavailable)
		return
	}
	location.Host = rloc.Host
	location.Path = req.URL.Path
	location.RawQuery = req.URL.Query().Encode()

	newReq, cancelFn := newRequestForProxy(location, req)
	defer cancelFn()

	if handlingInfo.proxyRoundTripper == nil {
		proxyError(w, req, "", http.StatusNotFound)
		return
	}

	proxyRoundTripper := handlingInfo.proxyRoundTripper
	upgrade := httpstream.IsUpgradeRequest(req)

	proxyRoundTripper = transport.NewAuthProxyRoundTripper(user.GetName(), user.GetGroups(), user.GetExtra(), proxyRoundTripper)

	// If we are upgrading, then the upgrade path tries to use this request with the TLS config we provide, but it does
	// NOT use the proxyRoundTripper.  It's a direct dial that bypasses the proxyRoundTripper.  This means that we have to
	// attach the "correct" user headers to the request ahead of time.
	if upgrade {
		transport.SetAuthProxyHeaders(newReq, user.GetName(), user.GetGroups(), user.GetExtra())
	}

	handler := proxy.NewUpgradeAwareHandler(location, proxyRoundTripper, true, upgrade, &responder{w: w})
	handler.InterceptRedirects = utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StreamingProxyRedirects)
	handler.RequireSameHostRedirects = utilfeature.DefaultFeatureGate.Enabled(genericfeatures.ValidateProxyRedirects)
	utilflowcontrol.RequestDelegated(req.Context())
	handler.ServeHTTP(w, newReq)
}
```