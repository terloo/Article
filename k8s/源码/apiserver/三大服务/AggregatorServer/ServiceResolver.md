# ServiceResolver
ServiceResolver是用于获取Service资源对象对应的url路径的接口

## 接口
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/resolvers.go
type ServiceResolver interface {
	ResolveEndpoint(namespace, name string, port int32) (*url.URL, error)
}
```
1. ResolveEndpoint：传入Service资源对象的namespace、名字、端口，获取该资源对象对应的url

## 实现类

### loopbackResolver
在特定的时候获取本地地址，其他时候交由委托ServieResolver进行处理
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/resolvers.go
func NewLoopbackServiceResolver(delegate ServiceResolver, host *url.URL) ServiceResolver {
	return &loopbackResolver{
		delegate: delegate,
		host:     host,
	}
}

type loopbackResolver struct {
	delegate ServiceResolver
	host     *url.URL
}

// 在传入namspace是default，name是kubernetes，port是443时，直接集群本地Service
func (r *loopbackResolver) ResolveEndpoint(namespace, name string, port int32) (*url.URL, error) {
	if namespace == "default" && name == "kubernetes" && port == 443 {
		return r.host, nil
	}
	return r.delegate.ResolveEndpoint(namespace, name, port)
}
```

### aggregatorClusterRouting
获取Service对应的ClusterIP
```go
// vendor/k8s.io/kube-aggregator/pkg/apiserver/resolvers.go
func NewClusterIPServiceResolver(services listersv1.ServiceLister) ServiceResolver {
	return &aggregatorClusterRouting{
		services: services,
	}
}

type aggregatorClusterRouting struct {
	services listersv1.ServiceLister
}

// 使用proxy.ResolveCluster来处理
func (r *aggregatorClusterRouting) ResolveEndpoint(namespace, name string, port int32) (*url.URL, error) {
	return proxy.ResolveCluster(r.services, namespace, name, port)
}
```

### aggregatorEndpointRouting
获取Service对应的所有EndPoint
```go
// vendor/k8s.io/kube-apiaggregator/pkg/apiserver/resolvers.go
func NewEndpointServiceResolver(services listersv1.ServiceLister, endpoints listersv1.EndpointsLister) ServiceResolver {
	return &aggregatorEndpointRouting{
		services:  services,
		endpoints: endpoints,
	}
}

type aggregatorEndpointRouting struct {
	services  listersv1.ServiceLister
	endpoints listersv1.EndpointsLister
}

// 使用proyx.ResolveEndpoint来处理
func (r *aggregatorEndpointRouting) ResolveEndpoint(namespace, name string, port int32) (*url.URL, error) {
	return proxy.ResolveEndpoint(r.services, r.endpoints, namespace, name, port)
}
```

## proxy.ResolveCluster
将
```go
// vendor/k8s.io/apiserver/pkg/util/proxy/proxy.go
func ResolveCluster(services listersv1.ServiceLister, namespace, id string, port int32) (*url.URL, error) {
    // 获取Service对象
	svc, err := services.Services(namespace).Get(id)
	if err != nil {
		return nil, err
	}

	switch {
	case svc.Spec.Type == v1.ServiceTypeClusterIP && svc.Spec.ClusterIP == v1.ClusterIPNone:
        // 如果是headlessService，则不能获取
		return nil, fmt.Errorf(`cannot route to service with ClusterIP "None"`)
	// use IP from a clusterIP for these service types
	case svc.Spec.Type == v1.ServiceTypeClusterIP, svc.Spec.Type == v1.ServiceTypeLoadBalancer, svc.Spec.Type == v1.ServiceTypeNodePort:
        // ClusterIP，LoadBalancer、NodePort。获取对应的ClusterIP
		svcPort, err := findServicePort(svc, port)
		if err != nil {
			return nil, err
		}
		return &url.URL{
			Scheme: "https",
			Host:   net.JoinHostPort(svc.Spec.ClusterIP, fmt.Sprintf("%d", svcPort.Port)),
		}, nil
	case svc.Spec.Type == v1.ServiceTypeExternalName:
        // ExternalName。直接拼接地址
		return &url.URL{
			Scheme: "https",
			Host:   net.JoinHostPort(svc.Spec.ExternalName, fmt.Sprintf("%d", port)),
		}, nil
	default:
		return nil, fmt.Errorf("unsupported service type %q", svc.Spec.Type)
	}
}
```

## proxy.ResolveEndpoint
```go
// vendor/k8s.io/apiserver/pkg/util/proxy/proxy.go
func ResolveEndpoint(services listersv1.ServiceLister, endpoints listersv1.EndpointsLister, namespace, id string, port int32) (*url.URL, error) {
	svc, err := services.Services(namespace).Get(id)
	if err != nil {
		return nil, err
	}

	svcPort, err := findServicePort(svc, port)
	if err != nil {
		return nil, err
	}

	switch {
	case svc.Spec.Type == v1.ServiceTypeClusterIP, svc.Spec.Type == v1.ServiceTypeLoadBalancer, svc.Spec.Type == v1.ServiceTypeNodePort:
		// these are fine
	default:
		return nil, fmt.Errorf("unsupported service type %q", svc.Spec.Type)
	}

	eps, err := endpoints.Endpoints(namespace).Get(svc.Name)
	if err != nil {
		return nil, err
	}
	if len(eps.Subsets) == 0 {
		return nil, errors.NewServiceUnavailable(fmt.Sprintf("no endpoints available for service %q", svc.Name))
	}

	// Pick a random Subset to start searching from.
	ssSeed := rand.Intn(len(eps.Subsets))

	// Find a Subset that has the port.
    // 选取一个随机的地址返回
	for ssi := 0; ssi < len(eps.Subsets); ssi++ {
		ss := &eps.Subsets[(ssSeed+ssi)%len(eps.Subsets)]
		if len(ss.Addresses) == 0 {
			continue
		}
		for i := range ss.Ports {
			if ss.Ports[i].Name == svcPort.Name {
				// Pick a random address.
				ip := ss.Addresses[rand.Intn(len(ss.Addresses))].IP
				port := int(ss.Ports[i].Port)
				return &url.URL{
					Scheme: "https",
					Host:   net.JoinHostPort(ip, strconv.Itoa(port)),
				}, nil
			}
		}
	}
	return nil, errors.NewServiceUnavailable(fmt.Sprintf("no endpoints available for service %q", id))
}
```