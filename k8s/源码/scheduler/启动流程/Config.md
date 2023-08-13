# Config
Config结构体保存着运行Kube-Scheduler所需要的所有参数和配置

## 结构体
```go
// cmd/kube-scheduler/app/config/config.go
// Config has all the context to run a Scheduler
type Config struct {
	// ComponentConfig is the scheduler server's configuration object.
	ComponentConfig kubeschedulerconfig.KubeSchedulerConfiguration

	// LoopbackClientConfig is a config for a privileged loopback connection
	LoopbackClientConfig *restclient.Config

	Authentication apiserver.AuthenticationInfo
	Authorization  apiserver.AuthorizationInfo
	SecureServing  *apiserver.SecureServingInfo

	Client             clientset.Interface
	KubeConfig         *restclient.Config
	InformerFactory    informers.SharedInformerFactory
	DynInformerFactory dynamicinformer.DynamicSharedInformerFactory

	//nolint:staticcheck // SA1019 this deprecated field still needs to be used for now. It will be removed once the migration is done.
	EventBroadcaster events.EventBroadcasterAdapter

	// LeaderElection is optional.
	LeaderElection *leaderelection.LeaderElectionConfig
}
```
1. ComponentConfig：配置资源对象

## CompletedConfig
表示该Config已经完成配置的结构体
```go
// cmd/kube-scheduler/app/config/config.go
type completedConfig struct {
	*Config
}

// CompletedConfig same as Config, just to swap private object.
type CompletedConfig struct {
	// Embed a private pointer that cannot be instantiated outside of this package.
	*completedConfig
}

// Complete fills in any fields not set that are required to have valid data. It's mutating the receiver.
func (c *Config) Complete() CompletedConfig {
	cc := completedConfig{c}

    // 验证一下apiService的身份和权限
	apiserver.AuthorizeClientBearerToken(c.LoopbackClientConfig, &c.Authentication, &c.Authorization)

	return CompletedConfig{&cc}
}
```
