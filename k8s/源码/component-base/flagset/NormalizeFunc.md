# NormalizeFunc
k8s提供了几个函数用于标准化选项名，方便绑定选项

## WordSepNormalizeFunc
```go
// k8s.io/component-base/cli/flag/flags.go
// 直接将选项名中的_当作-进行处理
func WordSepNormalizeFunc(f *pflag.FlagSet, name string) pflag.NormalizedName {
	if strings.Contains(name, "_") {
		return pflag.NormalizedName(strings.Replace(name, "_", "-", -1))
	}
	return pflag.NormalizedName(name)
}

// 选项名中的_当作-进行处理，并抛出警告
func WarnWordSepNormalizeFunc(f *pflag.FlagSet, name string) pflag.NormalizedName {
	if strings.Contains(name, "_") {
		nname := strings.Replace(name, "_", "-", -1)
		if _, alreadyWarned := underscoreWarnings[name]; !alreadyWarned {
			klog.Warningf("using an underscore in a flag name is not supported. %s has been converted to %s.", name, nname)
			underscoreWarnings[name] = true
		}

		return pflag.NormalizedName(nname)
	}
	return pflag.NormalizedName(name)
}
```