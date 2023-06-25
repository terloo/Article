# Config
GenericAPIServer的Config位于`vendor/k8s.io/apiserver/pkg/server/config.go`，该包一般被称为`genericapiserver`包

## 结构体
```go
// 字段大概是按照重要程度来进行排序的
type Config struct {
	// SecureServing is required to serve https
	SecureServing *SecureServingInfo

	// Authentication is the configuration for authentication
	Authentication AuthenticationInfo

	// Authorization is the configuration for authorization
	Authorization AuthorizationInfo

	// LoopbackClientConfig is a config for a privileged loopback connection to the API server
	// This is required for proper functioning of the PostStartHooks on a GenericAPIServer
	// TODO: move into SecureServing(WithLoopback) as soon as insecure serving is gone
	LoopbackClientConfig *restclient.Config

	// EgressSelector provides a lookup mechanism for dialing outbound connections.
	// It does so based on a EgressSelectorConfiguration which was read at startup.
	EgressSelector *egressselector.EgressSelector

	// RuleResolver is required to get the list of rules that apply to a given user
	// in a given namespace
	RuleResolver authorizer.RuleResolver
	// AdmissionControl performs deep inspection of a given request (including content)
	// to set values and determine whether its allowed
	AdmissionControl      admission.Interface
	CorsAllowedOriginList []string
	HSTSDirectives        []string
	// FlowControl, if not nil, gives priority and fairness to request handling
	FlowControl utilflowcontrol.Interface

	EnableIndex     bool
	EnableProfiling bool
	EnableDiscovery bool
	// Requires generic profiling enabled
	EnableContentionProfiling bool
	EnableMetrics             bool

	DisabledPostStartHooks sets.String
	// done values in this values for this map are ignored.
	PostStartHooks map[string]PostStartHookConfigEntry

	// Version will enable the /version endpoint if non-nil
	Version *version.Info
	// AuditBackend is where audit events are sent to.
	AuditBackend audit.Backend
	// AuditPolicyRuleEvaluator makes the decision of whether and how to audit log a request.
	AuditPolicyRuleEvaluator audit.PolicyRuleEvaluator
	// ExternalAddress is the host name to use for external (public internet) facing URLs (e.g. Swagger)
	// Will default to a value based on secure serving info and available ipv4 IPs.
	ExternalAddress string

	// TracerProvider can provide a tracer, which records spans for distributed tracing.
	TracerProvider *trace.TracerProvider

	//===========================================================================
	// Fields you probably don't care about changing
	//===========================================================================

	// BuildHandlerChainFunc allows you to build custom handler chains by decorating the apiHandler.
	BuildHandlerChainFunc func(apiHandler http.Handler, c *Config) (secure http.Handler)
	// HandlerChainWaitGroup allows you to wait for all chain handlers exit after the server shutdown.
	HandlerChainWaitGroup *utilwaitgroup.SafeWaitGroup
	// DiscoveryAddresses is used to build the IPs pass to discovery. If nil, the ExternalAddress is
	// always reported
	DiscoveryAddresses discovery.Addresses
	// The default set of healthz checks. There might be more added via AddHealthChecks dynamically.
	HealthzChecks []healthz.HealthChecker
	// The default set of livez checks. There might be more added via AddHealthChecks dynamically.
	LivezChecks []healthz.HealthChecker
	// The default set of readyz-only checks. There might be more added via AddReadyzChecks dynamically.
	ReadyzChecks []healthz.HealthChecker
	// LegacyAPIGroupPrefixes is used to set up URL parsing for authorization and for validating requests
	// to InstallLegacyAPIGroup. New API servers don't generally have legacy groups at all.
	LegacyAPIGroupPrefixes sets.String
	// RequestInfoResolver is used to assign attributes (used by admission and authorization) based on a request URL.
	// Use-cases that are like kubelets may need to customize this.
	RequestInfoResolver apirequest.RequestInfoResolver
	// Serializer is required and provides the interface for serializing and converting objects to and from the wire
	// The default (api.Codecs) usually works fine.
	Serializer runtime.NegotiatedSerializer
	// OpenAPIConfig will be used in generating OpenAPI spec. This is nil by default. Use DefaultOpenAPIConfig for "working" defaults.
	OpenAPIConfig *openapicommon.Config
	// SkipOpenAPIInstallation avoids installing the OpenAPI handler if set to true.
	SkipOpenAPIInstallation bool

	// RESTOptionsGetter is used to construct RESTStorage types via the generic registry.
	RESTOptionsGetter genericregistry.RESTOptionsGetter

	// If specified, all requests except those which match the LongRunningFunc predicate will timeout
	// after this duration.
	RequestTimeout time.Duration
	// If specified, long running requests such as watch will be allocated a random timeout between this value, and
	// twice this value.  Note that it is up to the request handlers to ignore or honor this timeout. In seconds.
	MinRequestTimeout int

	// This represents the maximum amount of time it should take for apiserver to complete its startup
	// sequence and become healthy. From apiserver's start time to when this amount of time has
	// elapsed, /livez will assume that unfinished post-start hooks will complete successfully and
	// therefore return true.
	LivezGracePeriod time.Duration
	// ShutdownDelayDuration allows to block shutdown for some time, e.g. until endpoints pointing to this API server
	// have converged on all node. During this time, the API server keeps serving, /healthz will return 200,
	// but /readyz will return failure.
	ShutdownDelayDuration time.Duration

	// The limit on the total size increase all "copy" operations in a json
	// patch may cause.
	// This affects all places that applies json patch in the binary.
	JSONPatchMaxCopyBytes int64
	// The limit on the request size that would be accepted and decoded in a write request
	// 0 means no limit.
	MaxRequestBodyBytes int64
	// MaxRequestsInFlight is the maximum number of parallel non-long-running requests. Every further
	// request has to wait. Applies only to non-mutating requests.
	MaxRequestsInFlight int
	// MaxMutatingRequestsInFlight is the maximum number of parallel mutating requests. Every further
	// request has to wait.
	MaxMutatingRequestsInFlight int
	// Predicate which is true for paths of long-running http requests
	LongRunningFunc apirequest.LongRunningRequestCheck

	// GoawayChance is the probability that send a GOAWAY to HTTP/2 clients. When client received
	// GOAWAY, the in-flight requests will not be affected and new requests will use
	// a new TCP connection to triggering re-balancing to another server behind the load balance.
	// Default to 0, means never send GOAWAY. Max is 0.02 to prevent break the apiserver.
	GoawayChance float64

	// MergedResourceConfig indicates which groupVersion enabled and its resources enabled/disabled.
	// This is composed of genericapiserver defaultAPIResourceConfig and those parsed from flags.
	// If not specify any in flags, then genericapiserver will only enable defaultAPIResourceConfig.
	MergedResourceConfig *serverstore.ResourceConfig

	// lifecycleSignals provides access to the various signals
	// that happen during lifecycle of the apiserver.
	// it's intentionally marked private as it should never be overridden.
	lifecycleSignals lifecycleSignals

	// StorageObjectCountTracker is used to keep track of the total number of objects
	// in the storage per resource, so we can estimate width of incoming requests.
	StorageObjectCountTracker flowcontrolrequest.StorageObjectCountTracker

	// ShutdownSendRetryAfter dictates when to initiate shutdown of the HTTP
	// Server during the graceful termination of the apiserver. If true, we wait
	// for non longrunning requests in flight to be drained and then initiate a
	// shutdown of the HTTP Server. If false, we initiate a shutdown of the HTTP
	// Server as soon as ShutdownDelayDuration has elapsed.
	// If enabled, after ShutdownDelayDuration elapses, any incoming request is
	// rejected with a 429 status code and a 'Retry-After' response.
	ShutdownSendRetryAfter bool

	//===========================================================================
	// values below here are targets for removal
	//===========================================================================

	// PublicAddress is the IP address where members of the cluster (kubelet,
	// kube-proxy, services, etc.) can reach the GenericAPIServer.
	// If nil or 0.0.0.0, the host's default interface will be used.
	PublicAddress net.IP

	// EquivalentResourceRegistry provides information about resources equivalent to a given resource,
	// and the kind associated with a given resource. As resources are installed, they are registered here.
	EquivalentResourceRegistry runtime.EquivalentResourceRegistry

	// APIServerID is the ID of this API server
	APIServerID string

	// StorageVersionManager holds the storage versions of the API resources installed by this server.
	StorageVersionManager storageversion.Manager
}
```
1. SecureServing：安全服务相关信息(证书等)

## 构造函数
```go
func NewConfig(codecs serializer.CodecFactory) *Config {
	defaultHealthChecks := []healthz.HealthChecker{healthz.PingHealthz, healthz.LogHealthz}
	var id string
	// 以uuid为底，随机一个id作为名字
	if feature.DefaultFeatureGate.Enabled(features.APIServerIdentity) {
		id = "kube-apiserver-" + uuid.New().String()
	}
	lifecycleSignals := newLifecycleSignals()

	return &Config{
		Serializer:                  codecs,
		// 通过Config构建GenericAPIServer时的HandlerChain，会在CompletedConfig.New()时执行
		BuildHandlerChainFunc:       DefaultBuildHandlerChain,
		HandlerChainWaitGroup:       new(utilwaitgroup.SafeWaitGroup),
		LegacyAPIGroupPrefixes:      sets.NewString(DefaultLegacyAPIPrefix),
		DisabledPostStartHooks:      sets.NewString(),
		PostStartHooks:              map[string]PostStartHookConfigEntry{},
		HealthzChecks:               append([]healthz.HealthChecker{}, defaultHealthChecks...),
		ReadyzChecks:                append([]healthz.HealthChecker{}, defaultHealthChecks...),
		LivezChecks:                 append([]healthz.HealthChecker{}, defaultHealthChecks...),
		EnableIndex:                 true,
		EnableDiscovery:             true,
		EnableProfiling:             true,
		EnableMetrics:               true,
		MaxRequestsInFlight:         400,
		MaxMutatingRequestsInFlight: 200,
		RequestTimeout:              time.Duration(60) * time.Second,
		MinRequestTimeout:           1800,
		LivezGracePeriod:            time.Duration(0),
		ShutdownDelayDuration:       time.Duration(0),
		// 1.5MB is the default client request size in bytes
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest size
		// increase the "copy" operations in a json patch may cause.
		JSONPatchMaxCopyBytes: int64(3 * 1024 * 1024),
		// 1.5MB is the recommended client request size in byte
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest request
		// body size to be accepted and decoded in a write request.
		MaxRequestBodyBytes: int64(3 * 1024 * 1024),

		// Default to treating watch as a long-running operation
		// Generic API servers have no inherent long-running subresources
		LongRunningFunc:           genericfilters.BasicLongRunningRequestCheck(sets.NewString("watch"), sets.NewString()),
		lifecycleSignals:          lifecycleSignals,
		StorageObjectCountTracker: flowcontrolrequest.NewStorageObjectCountTracker(lifecycleSignals.ShutdownInitiated.Signaled()),

		APIServerID:           id,
		StorageVersionManager: storageversion.NewDefaultManager(),
	}
}
```

## CompletedConfig
CompletedConfig是Config在调用完Complete()方法后生成的Config包装类，用以标识Config
```go
type CompletedConfig struct {
	// Embed a private pointer that cannot be instantiated outside of this package.
	// 使用这种方式，禁止在外部形成CompletedConfig
	*completedConfig
}

type completedConfig struct {
	*Config

	// SharedInformerFactory provides shared informers for resources
	SharedInformerFactory informers.SharedInformerFactory
}
```

## Complete()
```go
// vendor/k8s.io/apiserver/pkg/server/config.go
// 需要传入一个informerFactory
func (c *Config) Complete(informers informers.SharedInformerFactory) CompletedConfig {
	// 如果没有指定外部(公网)访问地址，则使用集群组件访问地址
	if len(c.ExternalAddress) == 0 && c.PublicAddress != nil {
		c.ExternalAddress = c.PublicAddress.String()
	}

	// if there is no port, and we listen on one securely, use that one
	// 如果公网访问地址没有端口，则使用SecureServing的端口
	if _, _, err := net.SplitHostPort(c.ExternalAddress); err != nil {
		if c.SecureServing == nil {
			klog.Fatalf("cannot derive external address port without listening on a secure port.")
		}
		_, port, err := c.SecureServing.HostPort()
		if err != nil {
			klog.Fatalf("cannot derive external address from the secure port: %v", err)
		}
		c.ExternalAddress = net.JoinHostPort(c.ExternalAddress, strconv.Itoa(port))
	}

	// 初始化openapi配置
	if c.OpenAPIConfig != nil {
		if c.OpenAPIConfig.SecurityDefinitions != nil {
			// Setup OpenAPI security: all APIs will have the same authentication for now.
			c.OpenAPIConfig.DefaultSecurity = []map[string][]string{}
			keys := []string{}
			for k := range *c.OpenAPIConfig.SecurityDefinitions {
				keys = append(keys, k)
			}
			sort.Strings(keys)
			for _, k := range keys {
				c.OpenAPIConfig.DefaultSecurity = append(c.OpenAPIConfig.DefaultSecurity, map[string][]string{k: {}})
			}
			if c.OpenAPIConfig.CommonResponses == nil {
				c.OpenAPIConfig.CommonResponses = map[int]spec.Response{}
			}
			if _, exists := c.OpenAPIConfig.CommonResponses[http.StatusUnauthorized]; !exists {
				c.OpenAPIConfig.CommonResponses[http.StatusUnauthorized] = spec.Response{
					ResponseProps: spec.ResponseProps{
						Description: "Unauthorized",
					},
				}
			}
		}

		// make sure we populate info, and info.version, if not manually set
		if c.OpenAPIConfig.Info == nil {
			c.OpenAPIConfig.Info = &spec.Info{}
		}
		if c.OpenAPIConfig.Info.Version == "" {
			if c.Version != nil {
				c.OpenAPIConfig.Info.Version = strings.Split(c.Version.String(), "-")[0]
			} else {
				c.OpenAPIConfig.Info.Version = "unversioned"
			}
		}
	}
	if c.DiscoveryAddresses == nil {
		c.DiscoveryAddresses = discovery.DefaultAddresses{DefaultAddress: c.ExternalAddress}
	}

	// 添加一个名为system:apiserver的用户，用以自访问
	AuthorizeClientBearerToken(c.LoopbackClientConfig, &c.Authentication, &c.Authorization)

	// 初始化api解析器
	if c.RequestInfoResolver == nil {
		c.RequestInfoResolver = NewRequestInfoResolver(c)
	}

	if c.EquivalentResourceRegistry == nil {
		if c.RESTOptionsGetter == nil {
			c.EquivalentResourceRegistry = runtime.NewEquivalentResourceRegistry()
		} else {
			c.EquivalentResourceRegistry = runtime.NewEquivalentResourceRegistryWithIdentity(func(groupResource schema.GroupResource) string {
				// use the storage prefix as the key if possible
				if opts, err := c.RESTOptionsGetter.GetRESTOptions(groupResource); err == nil {
					return opts.ResourcePrefix
				}
				// otherwise return "" to use the default key (parent GV name)
				return ""
			})
		}
	}

	// 包装为CompletedConfig
	return CompletedConfig{&completedConfig{c, informers}}
}
```