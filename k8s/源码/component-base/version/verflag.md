# verflag
用于处理所有组件的--version选项

## versionValue
自定义的flag类型，为了处理--version=raw的情况
```go
// vendor/k8s.io/component-base/version/verflag/verflag.go
type versionValue int

const (
	VersionFalse versionValue = 0
	VersionTrue  versionValue = 1
	VersionRaw   versionValue = 2
)

const strRawVersion string = "raw"

// IsBoolFlag为true的flag在命令行中可以省略=true
func (v *versionValue) IsBoolFlag() bool {
	return true
}

func (v *versionValue) Get() interface{} {
	return versionValue(*v)
}

// 赋值
func (v *versionValue) Set(s string) error {
    // 判断传入的值是否为raw
	if s == strRawVersion {
		*v = VersionRaw
		return nil
	}
    // 否则解析为boolean类型，然后转为versionValue类型
	boolVal, err := strconv.ParseBool(s)
	if boolVal {
		*v = VersionTrue
	} else {
		*v = VersionFalse
	}
	return err
}

func (v *versionValue) String() string {
	if *v == VersionRaw {
		return strRawVersion
	}
	return fmt.Sprintf("%v", bool(*v == VersionTrue))
}

// The type of the flag as required by the pflag.Value interface
func (v *versionValue) Type() string {
	return "version"
}
```

## flag
创建flag，处理默认值
1. 显示声明--version=xxx：使用xxx
2. 显示声明--version：使用true
3. 不显示声明--version：使用false
```go
// 创建一个flag，
func VersionVar(p *versionValue, name string, value versionValue, usage string) {
	// 不显示声明flag时的值
	*p = value
	flag.Var(p, name, usage) // 显示声明flag，并赋值时将值应用到p中
	// "--version" will be treated as "--version=true"
	// 显示声明flag，但不赋值的值
	flag.Lookup(name).NoOptDefVal = "true"
}

func Version(name string, value versionValue, usage string) *versionValue {
	p := new(versionValue)
	VersionVar(p, name, value, usage)
	return p
}

const versionFlagName = "version"

var (
	versionFlag = Version(versionFlagName, VersionFalse, "Print version information and quit")
	programName = "Kubernetes"
)

// AddFlags registers this package's flags on arbitrary FlagSets, such that they point to the
// same value as the global flags.
func AddFlags(fs *flag.FlagSet) {
	// 调用此函数，将--version的flag添加到flag集合中
	fs.AddFlag(flag.Lookup(versionFlagName))
}
```

## PrintAndExitIfRequested
检查--version，在false时不做处理，其余时候打印version信息然后退出程序
```go
// k8s.io/component-base/version/verflag/verflag.go
func PrintAndExitIfRequested() {
	if *versionFlag == VersionRaw {
		fmt.Printf("%#v\n", version.Get())
		os.Exit(0)
	} else if *versionFlag == VersionTrue {
		fmt.Printf("%s %s\n", programName, version.Get())
		os.Exit(0)
	}
}
```