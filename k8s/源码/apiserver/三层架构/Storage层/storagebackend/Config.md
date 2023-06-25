# Config
Config是供storagebackend函数所使用的，用于构建store实例的配置类

## 结构体
```go
// k8s.io/apiserver/pkg/storage/storagebackend/config.go
type TransportConfig struct {
	// ServerList is the list of storage servers to connect with.
	ServerList []string
	// TLS credentials
	KeyFile       string
	CertFile      string
	TrustedCAFile string
	// function to determine the egress dialer. (i.e. konnectivity server dialer)
	EgressLookup egressselector.Lookup
	// The TracerProvider can add tracing the connection
	TracerProvider *trace.TracerProvider
}

type Config struct {
	// Type defines the type of storage backend. Default ("") is "etcd3".
	Type string
	// Prefix is the prefix to all keys passed to storage.Interface methods.
	Prefix string
	// Transport holds all connection related info, i.e. equal TransportConfig means equal servers we talk to.
	Transport TransportConfig
	// Paging indicates whether the server implementation should allow paging (if it is
	// supported). This is generally configured by feature gating, or by a specific
	// resource type not wishing to allow paging, and is not intended for end users to
	// set.
	Paging bool

	Codec runtime.Codec
	// EncodeVersioner is the same groupVersioner used to build the
	// storage encoder. Given a list of kinds the input object might belong
	// to, the EncodeVersioner outputs the gvk the object will be
	// converted to before persisted in etcd.
	EncodeVersioner runtime.GroupVersioner
	// Transformer allows the value to be transformed prior to persisting into etcd.
	Transformer value.Transformer

	// CompactionInterval is an interval of requesting compaction from apiserver.
	// If the value is 0, no compaction will be issued.
	CompactionInterval time.Duration
	// CountMetricPollPeriod specifies how often should count metric be updated
	CountMetricPollPeriod time.Duration
	// DBMetricPollInterval specifies how often should storage backend metric be updated.
	DBMetricPollInterval time.Duration
	// HealthcheckTimeout specifies the timeout used when checking health
	HealthcheckTimeout time.Duration

	LeaseManagerConfig etcd3.LeaseManagerConfig

	// StorageObjectCountTracker is used to keep track of the total
	// number of objects in the storage per resource.
	StorageObjectCountTracker flowcontrolrequest.StorageObjectCountTracker
}
```

## 构造函数
使用Config的默认设置来构造Config，Config的其余值会被直接绑定到Options中
```go
// k8s.io/apiserver/pkg/storage/storagebackend/config.go
const (
	DefaultCompactInterval      = 5 * time.Minute
	DefaultDBMetricPollInterval = 30 * time.Second
	DefaultHealthcheckTimeout   = 2 * time.Second
)

// 需要传入一个codec和prefix
func NewDefaultConfig(prefix string, codec runtime.Codec) *Config {
	return &Config{
		Paging:               true,
		Prefix:               prefix,
		Codec:                codec,
		CompactionInterval:   DefaultCompactInterval,
		DBMetricPollInterval: DefaultDBMetricPollInterval,
		HealthcheckTimeout:   DefaultHealthcheckTimeout,
		LeaseManagerConfig:   etcd3.NewDefaultLeaseManagerConfig(),
	}
}
```

## ForResource
拷贝一份基础Config，并将其与GR绑定称为一个ConfigForResource
```go
// k8s.io/apiserver/pkg/storage/storagebackend/config.go
func (config *Config) ForResource(resource schema.GroupResource) *ConfigForResource {
	return &ConfigForResource{
		Config:        *config,
		GroupResource: resource,
	}
}
```