# PathOptions
kubelet中用于绑定命令行选项的结构体

## 实现类
```go
// k8s.io/clinet-go/tools/clientcmd/config.go
type PathOptions struct {
	// GlobalFile is the full path to the file to load as the global (final) option
	GlobalFile string
	// EnvVar is the env var name that points to the list of kubeconfig files to load
	EnvVar string
	// ExplicitFileFlag is the name of the flag to use for prompting for the kubeconfig file
	ExplicitFileFlag string

	// GlobalFileSubpath is an optional value used for displaying help
	GlobalFileSubpath string

	LoadingRules *ClientConfigLoadingRules
}
```
1. GlobalFile：推荐的kubeconfig文件位置
2. EnvVar：推荐的环境变量名
3. ExplicitFileFlag：推荐的命令行选项名
4. LoadingRules：底层使用ClientConfigLoadingRules进行实现

## 构建函数
构建并填充默认值
```go
// k8s.io/client-go/tools/clientcmd/config.go
func NewDefaultPathOptions() *PathOptions {
	ret := &PathOptions{
		GlobalFile:       RecommendedHomeFile,
		EnvVar:           RecommendedConfigPathEnvVar,
		ExplicitFileFlag: RecommendedConfigPathFlag,

		GlobalFileSubpath: path.Join(RecommendedHomeDir, RecommendedFileName),

		LoadingRules: NewDefaultClientConfigLoadingRules(),
	}
	ret.LoadingRules.DoNotResolvePaths = true

	return ret
}
```

## 绑定options
将选项名pathOptions.ExplicitFileFlag绑定到字段pathOptions.LoadingRules.ExplicitPath
```go
// k8s.io/kubeclt/pkg/cmd/config/config.go
cmd.PersistentFlags().StringVar(&pathOptions.LoadingRules.ExplicitPath, pathOptions.ExplicitFileFlag, pathOptions.LoadingRules.ExplicitPath, "use a particular kubeconfig file")
```

## GetLoadingPrecedence
```go
func (o *PathOptions) GetLoadingPrecedence() []string {
	if o.IsExplicitFile() {
		return []string{o.GetExplicitFile()}
	}

    // 没显示指定文件，则使用环境变量中的文件配置
	if envVarFiles := o.GetEnvVarFiles(); len(envVarFiles) > 0 {
		return envVarFiles
	}

    // 环境变量中也没配置，则使用推荐配置
	return []string{o.GlobalFile}
}

func (o *PathOptions) GetEnvVarFiles() []string {
	if len(o.EnvVar) == 0 {
		return []string{}
	}

	envVarValue := os.Getenv(o.EnvVar)
	if len(envVarValue) == 0 {
		return []string{}
	}

	fileList := filepath.SplitList(envVarValue)
	// prevent the same path load multiple times
	return deduplicate(fileList)
}
```

## GetStartingConfig
```go
// k8s.io/client-go/tools/clientcmd/config.go
func (o *PathOptions) GetStartingConfig() (*clientcmdapi.Config, error) {
	// don't mutate the original
	loadingRules := *o.LoadingRules
    // 重新配置其中的loadingRules
	loadingRules.Precedence = o.GetLoadingPrecedence()

    // 生成DeferredLoadingClientConfig，再生成配置
	clientConfig := NewNonInteractiveDeferredLoadingClientConfig(&loadingRules, &ConfigOverrides{})
	rawConfig, err := clientConfig.RawConfig()
	if os.IsNotExist(err) {
		return clientcmdapi.NewConfig(), nil
	}
	if err != nil {
		return nil, err
	}

	return &rawConfig, nil
}
```

## GetDefaultFilename
```go
// k8s.io/client-go/tools/clientcmd/config.go
func (o *PathOptions) GetDefaultFilename() string {
	if o.IsExplicitFile() {
		return o.GetExplicitFile()
	}

	if envVarFiles := o.GetEnvVarFiles(); len(envVarFiles) > 0 {
		if len(envVarFiles) == 1 {
			return envVarFiles[0]
		}

		// if any of the envvar files already exists, return it
		for _, envVarFile := range envVarFiles {
			if _, err := os.Stat(envVarFile); err == nil {
				return envVarFile
			}
		}

		// otherwise, return the last one in the list
		return envVarFiles[len(envVarFiles)-1]
	}

	return o.GlobalFile
}
```

## IsExplicitFile GetExplicitFile
```go
// k8s.io/client-go/tools/clientcmd/config.go
func (o *PathOptions) IsExplicitFile() bool {
	return len(o.LoadingRules.ExplicitPath) > 0
}

func (o *PathOptions) GetExplicitFile() string {
	return o.LoadingRules.ExplicitPath
}
```