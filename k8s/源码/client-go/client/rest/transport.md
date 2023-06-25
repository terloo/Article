# transport
transport存放着与`k8s.io/client-go/transport`库向交互的函数

## HTTPClientFor
通过restclient.Config获取http.Client的函数
```go
// k8s.io/client-go/rest/transport.go
func HTTPClientFor(config *Config) (*http.Client, error) {
	transport, err := TransportFor(config)
	if err != nil {
		return nil, err
	}
	var httpClient *http.Client
	if transport != http.DefaultTransport || config.Timeout > 0 {
		httpClient = &http.Client{
			Transport: transport,
			Timeout:   config.Timeout,
		}
	} else {
		httpClient = http.DefaultClient
	}

	return httpClient, nil
}

func TransportFor(config *Config) (http.RoundTripper, error) {
	cfg, err := config.TransportConfig()
	if err != nil {
		return nil, err
	}
    // 调用k8s.io/client-go/transport库的方法生成transport
	return transport.New(cfg)
}
```

## TransportConfig
为restclient.Config添加的一个获取transport.Config的方法
```go
// k8s.io/client-go/rest/transport.go
func (c *Config) TransportConfig() (*transport.Config, error) {
    // 构建一个transport.Config，并把restclient.Config中的值赋值
	conf := &transport.Config{
		UserAgent:          c.UserAgent,
		Transport:          c.Transport,
		WrapTransport:      c.WrapTransport,
		DisableCompression: c.DisableCompression,
		TLS: transport.TLSConfig{
			Insecure:   c.Insecure,
			ServerName: c.ServerName,
			CAFile:     c.CAFile,
			CAData:     c.CAData,
			CertFile:   c.CertFile,
			CertData:   c.CertData,
			KeyFile:    c.KeyFile,
			KeyData:    c.KeyData,
			NextProtos: c.NextProtos,
		},
		Username:        c.Username,
		Password:        c.Password,
		BearerToken:     c.BearerToken,
		BearerTokenFile: c.BearerTokenFile,
		Impersonate: transport.ImpersonationConfig{
			UserName: c.Impersonate.UserName,
			UID:      c.Impersonate.UID,
			Groups:   c.Impersonate.Groups,
			Extra:    c.Impersonate.Extra,
		},
		Dial:  c.Dial,
		Proxy: c.Proxy,
	}

	if c.ExecProvider != nil && c.AuthProvider != nil {
		return nil, errors.New("execProvider and authProvider cannot be used in combination")
	}

    // 两种验证提供器
	if c.ExecProvider != nil {
		var cluster *clientauthentication.Cluster
		if c.ExecProvider.ProvideClusterInfo {
			var err error
			cluster, err = ConfigToExecCluster(c)
			if err != nil {
				return nil, err
			}
		}
		provider, err := exec.GetAuthenticator(c.ExecProvider, cluster)
		if err != nil {
			return nil, err
		}
		if err := provider.UpdateTransportConfig(conf); err != nil {
			return nil, err
		}
	}
	if c.AuthProvider != nil {
		provider, err := GetAuthProvider(c.Host, c.AuthProvider, c.AuthConfigPersister)
		if err != nil {
			return nil, err
		}
		conf.Wrap(provider.WrapTransport)
	}
	return conf, nil
}
```