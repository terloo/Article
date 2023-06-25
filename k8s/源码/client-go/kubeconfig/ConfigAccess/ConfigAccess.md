# ConfigAccess
ConfigAccess是一个用于获取kubeconfig文件相关信息的接口

## 实现
1. ClientConfigLoadingRules：实现了ClientConfigLoader接口，合并不同的kubeconfig文件。
2. PathOptions：kubelet中用于绑定命令行选项的结构体

## ConfigAccess
```go
// k8s.io/client-go/tools/clientcmd/config.go
type ConfigAccess interface {
	// GetLoadingPrecedence returns the slice of files that should be used for loading and inspecting the config
	GetLoadingPrecedence() []string
	// GetStartingConfig returns the config that subcommands should being operating against.  It may or may not be merged depending on loading rules
	GetStartingConfig() (*clientcmdapi.Config, error)
	// GetDefaultFilename returns the name of the file you should write into (create if necessary), if you're trying to create a new stanza as opposed to updating an existing one.
	GetDefaultFilename() string
	// IsExplicitFile indicates whether or not this command is interested in exactly one file.  This implementation only ever does that  via a flag, but implementations that handle local, global, and flags may have more
	IsExplicitFile() bool
	// GetExplicitFile returns the particular file this command is operating against.  This implementation only ever has one, but implementations that handle local, global, and flags may have more
	GetExplicitFile() string
}
```
1. GetLoadingPrecedence：返回一个存放着所有kubeconfig文件路径的切片
2. GetStartingConfig：返回当前程序应该使用的经过合并后的kubeconfig配置类
3. GetDefaultFilename：返回程序想持久化kubeconfig配置类时，应使用的文件路径
4. IsExplicitFile：用户是否显示的指定了某个文件
5. GetExplicitFile：显示指定的文件路径

## ClientConfigLoader
ClientConfigLoader是ConfigAccess的子接口，提供了一个重新加载本地kubeconfig文件并返回配置的接口
```go
// k8s.io/client-go/tools/clientcmd/config.go
type ClientConfigLoader interface {
	ConfigAccess
	// IsDefaultConfig returns true if the returned config matches the defaults.
	IsDefaultConfig(*restclient.Config) bool
	// Load returns the latest config
	Load() (*clientcmdapi.Config, error)
}
```
1. IsDefaultConfig：传入一个restclient.Config，判断是否为默认配置
2. Load：(在配置文件修改后)重新加载本地kubeconfig文件，并返回kubeconfig类