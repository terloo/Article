# storagebackend
storagebackend是用于创建store的函数封装，接收一个ConfigForResource对象，使用该对象构建出store实例

## 使用ConfigForResource来创建一个store实例
```go
// vendor/k8s.io/apiserver/pkg/storage/storagebackend/factory/factory.go
func Create(c storagebackend.ConfigForResource, newFunc func() runtime.Object) (storage.Interface, DestroyFunc, error) {
	switch c.Type {
        // 拒绝etcdv2客户端的创建
	case storagebackend.StorageTypeETCD2:
		return nil, nil, fmt.Errorf("%s is no longer a supported storage backend", c.Type)
	case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
		return newETCD3Storage(c, newFunc)
	default:
		return nil, nil, fmt.Errorf("unknown storage type: %s", c.Type)
	}
}

// vendor/k8s.io/apiserver/pkg/storage/storagebackend/factory/etcd3.go
func newETCD3Storage(c storagebackend.ConfigForResource, newFunc func() runtime.Object) (storage.Interface, DestroyFunc, error) {
	stopCompactor, err := startCompactorOnce(c.Transport, c.CompactionInterval)
	if err != nil {
		return nil, nil, err
	}

    // 构建一个etcd的*clientv3.Client
	client, err := newETCD3Client(c.Transport)
	if err != nil {
		stopCompactor()
		return nil, nil, err
	}

	stopDBSizeMonitor, err := startDBSizeMonitorPerEndpoint(client, c.DBMetricPollInterval)
	if err != nil {
		return nil, nil, err
	}

	var once sync.Once
	destroyFunc := func() {
		// we know that storage destroy funcs are called multiple times (due to reuse in subresources).
		// Hence, we only destroy once.
		// TODO: fix duplicated storage destroy calls higher level
		once.Do(func() {
			stopCompactor()
			stopDBSizeMonitor()
			client.Close()
		})
	}
	transformer := c.Transformer
	if transformer == nil {
		transformer = value.IdentityTransformer
	}

    // 封装为storage.Interface接口，实现类是store
	return etcd3.New(client, c.Codec, newFunc, c.Prefix, c.GroupResource, transformer, c.Paging, c.LeaseManagerConfig), destroyFunc, nil
}

func newETCD3Client(c storagebackend.TransportConfig) (*clientv3.Client, error) {
	tlsInfo := transport.TLSInfo{
		CertFile:      c.CertFile,
		KeyFile:       c.KeyFile,
		TrustedCAFile: c.TrustedCAFile,
	}
	tlsConfig, err := tlsInfo.ClientConfig()
	if err != nil {
		return nil, err
	}
	// NOTE: Client relies on nil tlsConfig
	// for non-secure connections, update the implicit variable
	if len(c.CertFile) == 0 && len(c.KeyFile) == 0 && len(c.TrustedCAFile) == 0 {
		tlsConfig = nil
	}
	networkContext := egressselector.Etcd.AsNetworkContext()
	var egressDialer utilnet.DialFunc
	if c.EgressLookup != nil {
		egressDialer, err = c.EgressLookup(networkContext)
		if err != nil {
			return nil, err
		}
	}
	dialOptions := []grpc.DialOption{
		grpc.WithBlock(), // block until the underlying connection is up
		// use chained interceptors so that the default (retry and backoff) interceptors are added.
		// otherwise they will be overwritten by the metric interceptor.
		//
		// these optional interceptors will be placed after the default ones.
		// which seems to be what we want as the metrics will be collected on each attempt (retry)
		grpc.WithChainUnaryInterceptor(grpcprom.UnaryClientInterceptor),
		grpc.WithChainStreamInterceptor(grpcprom.StreamClientInterceptor),
	}
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		tracingOpts := []otelgrpc.Option{
			otelgrpc.WithPropagators(traces.Propagators()),
		}
		if c.TracerProvider != nil {
			tracingOpts = append(tracingOpts, otelgrpc.WithTracerProvider(*c.TracerProvider))
		}
		// Even if there is no TracerProvider, the otelgrpc still handles context propagation.
		// See https://github.com/open-telemetry/opentelemetry-go/tree/main/example/passthrough
		dialOptions = append(dialOptions,
			grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor(tracingOpts...)),
			grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor(tracingOpts...)))
	}
	if egressDialer != nil {
		dialer := func(ctx context.Context, addr string) (net.Conn, error) {
			if strings.Contains(addr, "//") {
				// etcd client prior to 3.5 passed URLs to dialer, normalize to address
				u, err := url.Parse(addr)
				if err != nil {
					return nil, err
				}
				addr = u.Host
			}
			return egressDialer(ctx, "tcp", addr)
		}
		dialOptions = append(dialOptions, grpc.WithContextDialer(dialer))
	}
	cfg := clientv3.Config{
		DialTimeout:          dialTimeout,
		DialKeepAliveTime:    keepaliveTime,
		DialKeepAliveTimeout: keepaliveTimeout,
		DialOptions:          dialOptions,
		Endpoints:            c.ServerList,
		TLS:                  tlsConfig,
	}

    // 读取配置后创建客户端
	return clientv3.New(cfg)
}
```