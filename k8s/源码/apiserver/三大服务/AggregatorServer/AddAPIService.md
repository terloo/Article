# AddAPIService
在有新的APIService被持久化时，AggregatorServer会被调用AddAPIService()方法来生成代理handler

| uri                        | 匹配模型 | 目标服务              |
| -------------------------- | -------- | --------------------- |
| `/api`                     | 精准匹配 | proxyHandler          |
| `/api/`                    | 前缀匹配 | proxyHandler          |
| `/apis/<group>/<version>`  | 精准匹配 | proxyHandler          |
| `/apis/<group>/<version>/` | 前缀匹配 | proxyHandler          |
| `/apis/<group>`            | 精准匹配 | groupDiscoveryHandler |
| `/apis/<group>/`           | 精准匹配 | groupDiscoveryHandler |


1. `/apis/<group>/<version>`(精准匹配)
2. `/apis/<group>/<version>/`(前缀匹配)

## 方法
```go
func (s *APIAggregator) AddAPIService(apiService *v1.APIService) error {
	// if the proxyHandler already exists, it needs to be updated. The aggregation bits do not
	// since they are wired against listers because they require multiple resources to respond
    // 如果已经有存在的proxyHandlers
	if proxyHandler, exists := s.proxyHandlers[apiService.Name]; exists {
		proxyHandler.updateAPIService(apiService)
		if s.openAPIAggregationController != nil {
			s.openAPIAggregationController.UpdateAPIService(proxyHandler, apiService)
		}
		if s.openAPIV3AggregationController != nil {
			s.openAPIV3AggregationController.UpdateAPIService(proxyHandler, apiService)
		}
		return nil
	}

    // 构建该APIService的路径
	proxyPath := "/apis/" + apiService.Spec.Group + "/" + apiService.Spec.Version
	// v1. is a special case for the legacy API.  It proxies to a wider set of endpoints.
	if apiService.Name == legacyAPIServiceName {
		proxyPath = "/api"
	}

	// register the proxy handler
	proxyHandler := &proxyHandler{
		localDelegate:              s.delegateHandler,
		proxyCurrentCertKeyContent: s.proxyCurrentCertKeyContent,
		proxyTransport:             s.proxyTransport,
		serviceResolver:            s.serviceResolver,
		egressSelector:             s.egressSelector,
	}
	proxyHandler.updateAPIService(apiService)
	if s.openAPIAggregationController != nil {
		s.openAPIAggregationController.AddAPIService(proxyHandler, apiService)
	}
	if s.openAPIV3AggregationController != nil {
		s.openAPIV3AggregationController.AddAPIService(proxyHandler, apiService)
	}
    // 注册该GV的代理handler
	s.proxyHandlers[apiService.Name] = proxyHandler
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle(proxyPath, proxyHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandlePrefix(proxyPath+"/", proxyHandler)

	// if we're dealing with the legacy group, we're done here
	if apiService.Name == legacyAPIServiceName {
		return nil
	}

	// if we've already registered the path with the handler, we don't want to do it again.
	if s.handledGroups.Has(apiService.Spec.Group) {
		return nil
	}

    // 注册该G的Discovery服务
	// it's time to register the group aggregation endpoint
	groupPath := "/apis/" + apiService.Spec.Group
	groupDiscoveryHandler := &apiGroupHandler{
		codecs:    aggregatorscheme.Codecs,
		groupName: apiService.Spec.Group,
		lister:    s.lister,
		delegate:  s.delegateHandler,
	}
	// aggregation is protected
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle(groupPath, groupDiscoveryHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle(groupPath+"/", groupDiscoveryHandler)
	s.handledGroups.Insert(apiService.Spec.Group)
	return nil
}
```