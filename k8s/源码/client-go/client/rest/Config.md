# Config
Config是client-go中用于保存REST客户端配置的结构体

## 结构体
```go
// k8s.io/client-go/rest/config.go
type Config struct {
	// Host must be a host string, a host:port pair, or a URL to the base of the apiserver.
	// If a URL is given then the (optional) Path of that URL represents a prefix that must
	// be appended to all request URIs used to access the apiserver. This allows a frontend
	// proxy to easily relocate all of the apiserver endpoints.
	Host string
	// APIPath is a sub-path that points to an API root.
	APIPath string

	// ContentConfig contains settings that affect how objects are transformed when
	// sent to the server.
	ContentConfig

	// Server requires Basic authentication
	Username string
	Password string `datapolicy:"password"`

	// Server requires Bearer authentication. This client will not attempt to use
	// refresh tokens for an OAuth2 flow.
	// TODO: demonstrate an OAuth2 compatible client.
	BearerToken string `datapolicy:"token"`

	// Path to a file containing a BearerToken.
	// If set, the contents are periodically read.
	// The last successfully read value takes precedence over BearerToken.
	BearerTokenFile string

	// Impersonate is the configuration that RESTClient will use for impersonation.
	Impersonate ImpersonationConfig

	// Server requires plugin-specified authentication.
	AuthProvider *clientcmdapi.AuthProviderConfig

	// Callback to persist config for AuthProvider.
	AuthConfigPersister AuthProviderConfigPersister

	// Exec-based authentication provider.
	ExecProvider *clientcmdapi.ExecConfig

	// TLSClientConfig contains settings to enable transport layer security
	TLSClientConfig

	// UserAgent is an optional field that specifies the caller of this request.
	UserAgent string

	// DisableCompression bypasses automatic GZip compression requests to the
	// server.
	DisableCompression bool

	// Transport may be used for custom HTTP behavior. This attribute may not
	// be specified with the TLS client certificate options. Use WrapTransport
	// to provide additional per-server middleware behavior.
	Transport http.RoundTripper
	// WrapTransport will be invoked for custom HTTP behavior after the underlying
	// transport is initialized (either the transport created from TLSClientConfig,
	// Transport, or http.DefaultTransport). The config may layer other RoundTrippers
	// on top of the returned RoundTripper.
	//
	// A future release will change this field to an array. Use config.Wrap()
	// instead of setting this value directly.
	WrapTransport transport.WrapperFunc

	// QPS indicates the maximum QPS to the master from this client.
	// If it's zero, the created RESTClient will use DefaultQPS: 5
	QPS float32

	// Maximum burst for throttle.
	// If it's zero, the created RESTClient will use DefaultBurst: 10.
	Burst int

	// Rate limiter for limiting connections to the master from this client. If present overwrites QPS/Burst
	RateLimiter flowcontrol.RateLimiter

	// WarningHandler handles warnings in server responses.
	// If not set, the default warning handler is used.
	// See documentation for SetDefaultWarningHandler() for details.
	WarningHandler WarningHandler

	// The maximum length of time to wait before giving up on a server request. A value of zero means no timeout.
	Timeout time.Duration

	// Dial specifies the dial function for creating unencrypted TCP connections.
	Dial func(ctx context.Context, network, address string) (net.Conn, error)

	// Proxy is the proxy func to be used for all requests made by this
	// transport. If Proxy is nil, http.ProxyFromEnvironment is used. If Proxy
	// returns a nil *URL, no proxy is used.
	//
	// socks5 proxying does not currently support spdy streaming endpoints.
	Proxy func(*http.Request) (*url.URL, error)

	// Version forces a specific version to be used (if registered)
	// Do we need this?
	// Version string
}
```
1. Host：api-server的地址，host:port或者URL形式
2. APIPath：api路径地址，如/api，/apis
3. ContentConfig：定义了对象传输的数据格式
4. Username、Password：BasicAuth验证方式使用的字段
5. BearerToken：Token验证方式使用的字段
6. BearerTokenFile：如果设置了该字段，上下文会定期读取该目录，并用目录中的值代替BearerToken
7. Impersonate：配置客户端所使用的发起者配置(UserAgent头)
8. AuthProvider：插件式验证方式提供器，无法与ExecProvider共存
9. AuthConfigPersister：AuthProvider的缓存
10. ExecProvider：插件式验证方式提供器，无法与AuthProvider共存
11. TLSClientConfig：TSL证书和协议相关配置
12. UserAgent：UserAgent头
13. DisableCompression：是否关闭gzip
14. Transport：标准库http.RoundTripper的实现类，一般是标准库http.Transport类，用于自定http行为
15. WrapTransport：对Transport进行包装的函数，将会在Transport初始化完成后调用
16. QPS、Burst、RateLimiter：限流措施
17. WarningHandler：服务相应警告处理函数，不设置时，将会使用默认警告处理函数
18. Timeout：请求超时时间
19. Dial：建立非加密TCP连接的函数
20. Proxy：代理函数，传入一个请求，返回代理服务器的URL。如果不设置，使用标准库的http.ProxyFromEnvironment


## 其余结构体
```go
// k8s.io/client-go/rest/config.go
type ContentConfig struct {
	// AcceptContentTypes specifies the types the client will accept and is optional.
	// If not set, ContentType will be used to define the Accept header
	AcceptContentTypes string
	// ContentType specifies the wire format used to communicate with the server.
	// This value will be set as the Accept header on requests made to the server, and
	// as the default content type on any object sent to the server. If not set,
	// "application/json" is used.
	ContentType string
	// GroupVersion is the API version to talk to. Must be provided when initializing
	// a RESTClient directly. When initializing a Client, will be set with the default
	// code version.
	GroupVersion *schema.GroupVersion
	// NegotiatedSerializer is used for obtaining encoders and decoders for multiple
	// supported media types.
	//
	// TODO: NegotiatedSerializer will be phased out as internal clients are removed
	//   from Kubernetes.
	NegotiatedSerializer runtime.NegotiatedSerializer
}

type ImpersonationConfig struct {
	// UserName is the username to impersonate on each request.
	UserName string
	// UID is a unique value that identifies the user.
	UID string
	// Groups are the groups to impersonate on each request.
	Groups []string
	// Extra is a free-form field which can be used to link some authentication information
	// to authorization information.  This field allows you to impersonate it.
	Extra map[string][]string
}

type TLSClientConfig struct {
	// Server should be accessed without verifying the TLS certificate. For testing only.
	Insecure bool
	// ServerName is passed to the server for SNI and is used in the client to check server
	// certificates against. If ServerName is empty, the hostname used to contact the
	// server is used.
	ServerName string

	// Server requires TLS client certificate authentication
	CertFile string
	// Server requires TLS client certificate authentication
	KeyFile string
	// Trusted root certificates for server
	CAFile string

	// CertData holds PEM-encoded bytes (typically read from a client certificate file).
	// CertData takes precedence over CertFile
	CertData []byte
	// KeyData holds PEM-encoded bytes (typically read from a client certificate key file).
	// KeyData takes precedence over KeyFile
	KeyData []byte `datapolicy:"security-key"`
	// CAData holds PEM-encoded bytes (typically read from a root certificates bundle).
	// CAData takes precedence over CAFile
	CAData []byte

	// NextProtos is a list of supported application level protocols, in order of preference.
	// Used to populate tls.Config.NextProtos.
	// To indicate to the server http/1.1 is preferred over http/2, set to ["http/1.1", "h2"] (though the server is free to ignore that preference).
	// To use only http/1.1, set to ["http/1.1"].
	NextProtos []string
}
```

## 构建函数
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
func BuildConfigFromFlags(masterUrl, kubeconfigPath string) (*restclient.Config, error) {
    // 如果未指定masterUrl和kubeconfigPath，则认为客户端在Pod中，使用在集群中的配置
	if kubeconfigPath == "" && masterUrl == "" {
		klog.Warning("Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.")
		kubeconfig, err := restclient.InClusterConfig()
		if err == nil {
			return kubeconfig, nil
		}
		klog.Warning("error creating inClusterConfig, falling back to default config: ", err)
	}
    // 否则使用延迟加载ClientConfig
	return NewNonInteractiveDeferredLoadingClientConfig(
		&ClientConfigLoadingRules{ExplicitPath: kubeconfigPath},
		&ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: masterUrl}}).ClientConfig()
}
```

## 工具型函数

### DefaultKubernetesUserAgent
获取默认的KubernetesUserAgent
```go
// k8s.io/client-go/rest/config.go
func DefaultKubernetesUserAgent() string {
	return buildUserAgent(
		adjustCommand(os.Args[0]),
		adjustVersion(version.Get().GitVersion),
		gruntime.GOOS,
		gruntime.GOARCH,
		adjustCommit(version.Get().GitCommit))
}

func adjustCommit(c string) string {
	if len(c) == 0 {
		return "unknown"
	}
	if len(c) > 7 {
		return c[:7]
	}
	return c
}

func adjustVersion(v string) string {
	if len(v) == 0 {
		return "unknown"
	}
	seg := strings.SplitN(v, "-", 2)
	return seg[0]
}

func adjustCommand(p string) string {
	// Unlikely, but better than returning "".
	if len(p) == 0 {
		return "unknown"
	}
	return filepath.Base(p)
}

func buildUserAgent(command, version, os, arch, commit string) string {
	return fmt.Sprintf(
		"%s/%s (%s/%s) kubernetes/%s", command, version, os, arch, commit)
}
```

### InClusterConfig
在Pod中获取Config
```go
// k8s.io/client-go/rest/config.go
// InClusterConfig returns a config object which uses the service account
// kubernetes gives to pods. It's intended for clients that expect to be
// running inside a pod running on kubernetes. It will return ErrNotInCluster
// if called from a process not running in a kubernetes environment.
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

	return &Config{
		// TODO: switch to using cluster DNS.
		Host:            "https://" + net.JoinHostPort(host, port),
		TLSClientConfig: tlsClientConfig,
		BearerToken:     string(token),
		BearerTokenFile: tokenFile,
	}, nil
}
```

### AddUserAgent
向Config中添加UserAgent字段，为默认的UserAgent与自定义的userAgent的拼接
```go
// k8s.io/client-go/rest/config.go
func AddUserAgent(config *Config, userAgent string) *Config {
	fullUserAgent := DefaultKubernetesUserAgent() + "/" + userAgent
	config.UserAgent = fullUserAgent
	return config
}
```

### CopyConfig
拷贝config
```go
// k8s.io/client-go/rest/config.go
func CopyConfig(config *Config) *Config {
	c := &Config{
		Host:            config.Host,
		APIPath:         config.APIPath,
		ContentConfig:   config.ContentConfig,
		Username:        config.Username,
		Password:        config.Password,
		BearerToken:     config.BearerToken,
		BearerTokenFile: config.BearerTokenFile,
		Impersonate: ImpersonationConfig{
			UserName: config.Impersonate.UserName,
			UID:      config.Impersonate.UID,
			Groups:   config.Impersonate.Groups,
			Extra:    config.Impersonate.Extra,
		},
		AuthProvider:        config.AuthProvider,
		AuthConfigPersister: config.AuthConfigPersister,
		ExecProvider:        config.ExecProvider,
		TLSClientConfig: TLSClientConfig{
			Insecure:   config.TLSClientConfig.Insecure,
			ServerName: config.TLSClientConfig.ServerName,
			CertFile:   config.TLSClientConfig.CertFile,
			KeyFile:    config.TLSClientConfig.KeyFile,
			CAFile:     config.TLSClientConfig.CAFile,
			CertData:   config.TLSClientConfig.CertData,
			KeyData:    config.TLSClientConfig.KeyData,
			CAData:     config.TLSClientConfig.CAData,
			NextProtos: config.TLSClientConfig.NextProtos,
		},
		UserAgent:          config.UserAgent,
		DisableCompression: config.DisableCompression,
		Transport:          config.Transport,
		WrapTransport:      config.WrapTransport,
		QPS:                config.QPS,
		Burst:              config.Burst,
		RateLimiter:        config.RateLimiter,
		WarningHandler:     config.WarningHandler,
		Timeout:            config.Timeout,
		Dial:               config.Dial,
		Proxy:              config.Proxy,
	}
	if config.ExecProvider != nil && config.ExecProvider.Config != nil {
		c.ExecProvider.Config = config.ExecProvider.Config.DeepCopyObject()
	}
	return c
}
```

### AnonymousClientConfig
拷贝config，但是移除自定义自定义HTTP行为
```go
// k8s.io/client-go/rest/config.go
func AnonymousClientConfig(config *Config) *Config {
	// copy only known safe fields
	return &Config{
		Host:          config.Host,
		APIPath:       config.APIPath,
		ContentConfig: config.ContentConfig,
		TLSClientConfig: TLSClientConfig{
			Insecure:   config.Insecure,
			ServerName: config.ServerName,
			CAFile:     config.TLSClientConfig.CAFile,
			CAData:     config.TLSClientConfig.CAData,
			NextProtos: config.TLSClientConfig.NextProtos,
		},
		RateLimiter:        config.RateLimiter,
		WarningHandler:     config.WarningHandler,
		UserAgent:          config.UserAgent,
		DisableCompression: config.DisableCompression,
		QPS:                config.QPS,
		Burst:              config.Burst,
		Timeout:            config.Timeout,
		Dial:               config.Dial,
		Proxy:              config.Proxy,
	}
}
```