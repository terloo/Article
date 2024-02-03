# inClusterClientConfig
当程序运行在Pod中时，可以使用集群向Pod中注入的信息来生成restclient.Config

## 实现类
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
type inClusterClientConfig struct {
	overrides               *ConfigOverrides
	inClusterConfigProvider func() (*restclient.Config, error)
}
```
1. overrides：覆盖原本的配置
2. inClusterConfigProvider：从集群环境读取信息生成restclient.Config的函数。一般采用restcliet.InClusterConfig

## restclient.InClusterConfig
从集群环境读取信息生成restclient.Config的函数
```go
// k8s.io/client-go/rest/config.go
func InClusterConfig() (*Config, error) {
	const (
		tokenFile  = "/var/run/secrets/kubernetes.io/serviceaccount/token"
		rootCAFile = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
	)
	host, port := os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_SERVICE_PORT")
	if len(host) == 0 || len(port) == 0 {
		return nil, ErrNotInCluster
	}

	token, err := ioutil.ReadFile(tokenFile)
	if err != nil {
		return nil, err
	}

	tlsClientConfig := TLSClientConfig{}

	if _, err := certutil.NewPool(rootCAFile); err != nil {
		klog.Errorf("Expected to load root CA config from %s, but got err: %v", rootCAFile, err)
	} else {
		tlsClientConfig.CAFile = rootCAFile
	}

    // 使用Token的方式进行验证
	return &Config{
		// TODO: switch to using cluster DNS.
		Host:            "https://" + net.JoinHostPort(host, port),
		TLSClientConfig: tlsClientConfig,
		BearerToken:     string(token),
		BearerTokenFile: tokenFile,
	}, nil
}
```

## ClientConfig
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
func (config *inClusterClientConfig) ClientConfig() (*restclient.Config, error) {
	inClusterConfigProvider := config.inClusterConfigProvider
    // 如果没指定inClusterConfigProvider，则使用restclient.InClusterConfig
	if inClusterConfigProvider == nil {
		inClusterConfigProvider = restclient.InClusterConfig
	}

	icc, err := inClusterConfigProvider()
	if err != nil {
		return nil, err
	}

    // 覆盖原本的配置，只覆盖host,token,CA
	// in-cluster configs only takes a host, token, or CA file
	// if any of them were individually provided, overwrite anything else
	if config.overrides != nil {
		if server := config.overrides.ClusterInfo.Server; len(server) > 0 {
			icc.Host = server
		}
		if len(config.overrides.AuthInfo.Token) > 0 || len(config.overrides.AuthInfo.TokenFile) > 0 {
			icc.BearerToken = config.overrides.AuthInfo.Token
			icc.BearerTokenFile = config.overrides.AuthInfo.TokenFile
		}
		if certificateAuthorityFile := config.overrides.ClusterInfo.CertificateAuthority; len(certificateAuthorityFile) > 0 {
			icc.TLSClientConfig.CAFile = certificateAuthorityFile
		}
	}

	return icc, nil
}
```

## Possible
判断是否处于集群环境中
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
func (config *inClusterClientConfig) Possible() bool {
	fi, err := os.Stat("/var/run/secrets/kubernetes.io/serviceaccount/token")
	return os.Getenv("KUBERNETES_SERVICE_HOST") != "" &&
		os.Getenv("KUBERNETES_SERVICE_PORT") != "" &&
		err == nil && !fi.IsDir()
}
```