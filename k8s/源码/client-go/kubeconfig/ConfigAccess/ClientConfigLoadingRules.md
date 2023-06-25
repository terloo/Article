# ClientConfigLoadingRules
ConfigAccess接口的实现

## config读取规则
1. 如果用默认构造函数构造
   1. 如果显示声明了文件路径，则使用该文件，没有该文件返回error
   2. 读取环境变量KUBECONFIG，如果有，合并环境变量中的所有路径文件
   3. 使用文件路径`~/.kube/config`
   4. 返回空配置，无报错
2. 不使用默认构造函数构造
   1. 如果显示声明了文件路径，则使用该文件，没有该文件返回error
   2. 使用并合并Precedence字段中配置的文件路径
   3. 返回空配置，无报错

## 常量
```go
// k8s.io/client-go/tools/clientcmd/loader.go
const (
	RecommendedConfigPathFlag   = "kubeconfig"
	RecommendedConfigPathEnvVar = "KUBECONFIG"
	RecommendedHomeDir          = ".kube"
	RecommendedFileName         = "config"
	RecommendedSchemaName       = "schema"
)

var (
	RecommendedConfigDir  = filepath.Join(homedir.HomeDir(), RecommendedHomeDir)
	RecommendedHomeFile   = filepath.Join(RecommendedConfigDir, RecommendedFileName)
	RecommendedSchemaFile = filepath.Join(RecommendedConfigDir, RecommendedSchemaName)
)
```
1. RecommendedConfigDir：~/.kube
2. RecommendedHomeFile：~/.kube/config
3. RecommendedSchemaFile：~/.kube/schema
4. 推荐使用的环境变量：KUBECONFIG
5. 推荐使用的命令行选项名：kubeconfig

## 实现类
```go
// k8s.io/client-go/tools/clientcmd/loader.go
type ClientConfigLoadingRules struct {
	ExplicitPath string
	Precedence   []string

	// MigrationRules is a map of destination files to source files.  If a destination file is not present, then the source file is checked.
	// If the source file is present, then it is copied to the destination file BEFORE any further loading happens.
	MigrationRules map[string]string

	// DoNotResolvePaths indicates whether or not to resolve paths with respect to the originating files.  This is phrased as a negative so
	// that a default object that doesn't set this will usually get the behavior it wants.
	DoNotResolvePaths bool

	// DefaultClientConfig is an optional field indicating what rules to use to calculate a default configuration.
	// This should match the overrides passed in to ClientConfig loader.
	DefaultClientConfig ClientConfig

	// WarnIfAllMissing indicates whether the configuration files pointed by KUBECONFIG environment variable are present or not.
	// In case of missing files, it warns the user about the missing files.
	WarnIfAllMissing bool
}
```
1. ExplicitPath：显示指定的文件路径
2. Precedence：所有kubeconfig配置文件路径
3. MigrationRules：迁移规则，目标文件->源文件。将源文件复制到目标文件中

## 构造函数
该构造函数使用的值都是默认值，非必须调用
```go
// k8s.io/client-go/tools/clientcmd/loader.go
func NewDefaultClientConfigLoadingRules() *ClientConfigLoadingRules {
	chain := []string{}
	warnIfAllMissing := false

	envVarFiles := os.Getenv(RecommendedConfigPathEnvVar)
	if len(envVarFiles) != 0 {
        // 读取环境变量KUBECONFIG，使用操作系统的ListSepatator分隔符进行分割
		fileList := filepath.SplitList(envVarFiles)
		// prevent the same path load multiple times
        // 去重后添加到Precedence字段中
		chain = append(chain, deduplicate(fileList)...)
		warnIfAllMissing = true

	} else {
        // 如果环境变量中没有值，则使用默认的kubeconfig路径
		chain = append(chain, RecommendedHomeFile)
	}

	return &ClientConfigLoadingRules{
		Precedence:       chain,
        // 默认的迁移规则，用于处理旧版本文件名
		MigrationRules:   currentMigrationRules(),
		WarnIfAllMissing: warnIfAllMissing,
	}
}

// 默认的迁移规则，将旧的默认kubeconfig文件路径复制到新的kubeconfig文件路径
func currentMigrationRules() map[string]string {
	var oldRecommendedHomeFileName string
	if goruntime.GOOS == "windows" {
		oldRecommendedHomeFileName = RecommendedFileName
	} else {
		oldRecommendedHomeFileName = ".kubeconfig"
	}
	return map[string]string{
		RecommendedHomeFile: filepath.Join(os.Getenv("HOME"), RecommendedHomeDir, oldRecommendedHomeFileName),
	}
}
```

## Load
```go
// k8s.io/client-go/tools/clientcmd/loader.go
func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error) {
	// 处理迁移规则
	if err := rules.Migrate(); err != nil {
		return nil, err
	}

	errlist := []error{}
	missingList := []string{}

	kubeConfigFiles := []string{}

	// Make sure a file we were explicitly told to use exists
	// 如果显示声明了文件路径，则使用显示声明的文件路径
	// 否则，使用Precedence中所有文件的合并结果
	if len(rules.ExplicitPath) > 0 {
		if _, err := os.Stat(rules.ExplicitPath); os.IsNotExist(err) {
			return nil, err
		}
		kubeConfigFiles = append(kubeConfigFiles, rules.ExplicitPath)

	} else {
		kubeConfigFiles = append(kubeConfigFiles, rules.Precedence...)
	}

	// 分别加载所有kubeconfig文件
	kubeconfigs := []*clientcmdapi.Config{}
	// read and cache the config files so that we only look at them once
	for _, filename := range kubeConfigFiles {
		if len(filename) == 0 {
			// no work to do
			continue
		}

		// 解码文件，并将其转为内部版本，并给Context的LocationOfOrigin字段赋值
		config, err := LoadFromFile(filename)

		if os.IsNotExist(err) {
			// skip missing files
			// Add to the missing list to produce a warning
			missingList = append(missingList, filename)
			continue
		}

		if err != nil {
			errlist = append(errlist, fmt.Errorf("error loading config file \"%s\": %v", filename, err))
			continue
		}

		kubeconfigs = append(kubeconfigs, config)
	}

	if rules.WarnIfAllMissing && len(missingList) > 0 && len(kubeconfigs) == 0 {
		klog.Warningf("Config not found: %s", strings.Join(missingList, ", "))
	}

	// first merge all of our maps
	// 先以正序合并所有kubeconfig
	mapConfig := clientcmdapi.NewConfig()

	for _, kubeconfig := range kubeconfigs {
		mergo.Merge(mapConfig, kubeconfig, mergo.WithOverride)
	}

	// merge all of the struct values in the reverse order so that priority is given correctly
	// errors are not added to the list the second time
	// 再以逆序合并所有kubeconfig
	nonMapConfig := clientcmdapi.NewConfig()
	for i := len(kubeconfigs) - 1; i >= 0; i-- {
		kubeconfig := kubeconfigs[i]
		mergo.Merge(nonMapConfig, kubeconfig, mergo.WithOverride)
	}

	// 再用逆序合并结果覆盖正序合并结果
	// since values are overwritten, but maps values are not, we can merge the non-map config on top of the map config and
	// get the values we expect.
	config := clientcmdapi.NewConfig()
	mergo.Merge(config, mapConfig, mergo.WithOverride)
	mergo.Merge(config, nonMapConfig, mergo.WithOverride)

	if rules.ResolvePaths() {
		// 如果需要，将文件地址转化为绝对路径
		if err := ResolveLocalPaths(config); err != nil {
			errlist = append(errlist, err)
		}
	}
	return config, utilerrors.NewAggregate(errlist)
}
```

## GetLoadingPrecedence
实现ConfigAccess
```go
// k8s.io/client-go/tools/clientcmd/loader.go
func (rules *ClientConfigLoadingRules) GetLoadingPrecedence() []string {
	// 如果显示指定了文件，那么只使用该显示指定的文件
	if len(rules.ExplicitPath) > 0 {
		return []string{rules.ExplicitPath}
	}

	return rules.Precedence
}
```

## GetStartingConfig
实现ConfigAccess
```go
// k8s.io/client-go/tools/clientcmd/loader.go
func (rules *ClientConfigLoadingRules) GetStartingConfig() (*clientcmdapi.Config, error) {
	clientConfig := NewNonInteractiveDeferredLoadingClientConfig(rules, &ConfigOverrides{})
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
实现ConfigAccess
```go
// k8s.io/client-go/tools/clientcmd/loader.go
func (rules *ClientConfigLoadingRules) GetDefaultFilename() string {
	// Explicit file if we have one.
	if rules.IsExplicitFile() {
		return rules.GetExplicitFile()
	}
	// Otherwise, first existing file from precedence.
	// 如果没有显示指定文件，则使用kubeconfig文件列表中第一个
	for _, filename := range rules.GetLoadingPrecedence() {
		if _, err := os.Stat(filename); err == nil {
			return filename
		}
	}
	// If none exists, use the first from precedence.
	if len(rules.Precedence) > 0 {
		return rules.Precedence[0]
	}
	return ""
}
```

## IsExplicitFile  GetExplicitFile
```go
// k8s.io/client-go/tools/clientcmd/loader.go
func (rules *ClientConfigLoadingRules) IsExplicitFile() bool {
	return len(rules.ExplicitPath) > 0
}

func (rules *ClientConfigLoadingRules) GetExplicitFile() string {
	return rules.ExplicitPath
}
```

## IsDefaultConfig
```go
// k8s.io/client-go/tools/clientcmd/loader.gogo
func (rules *ClientConfigLoadingRules) IsDefaultConfig(config *restclient.Config) bool {
	if rules.DefaultClientConfig == nil {
		return false
	}
	defaultConfig, err := rules.DefaultClientConfig.ClientConfig()
	if err != nil {
		return false
	}
	return reflect.DeepEqual(config, defaultConfig)
}
```