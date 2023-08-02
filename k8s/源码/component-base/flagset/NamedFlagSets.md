# NamedFlagSets
NamedFlagSets是对cobra中flagset的封装，可以将不同的命令行选项进行分组命名，并保持有序。在查看帮助命令时，可以以更好的格式进行输出

## 结构体
```go
// k8s.io/component-base/cli/flag/sectioned.go
type NamedFlagSets struct {
    // 按顺序排列的flag集合名
	// Order is an ordered list of flag set names.
	Order []string
    // 保存flag集合，按名字进行索引
	// FlagSets stores the flag sets by name.
	FlagSets map[string]*pflag.FlagSet
    // 用于标准化选项名的函数
	// NormalizeNameFunc is the normalize function which used to initialize FlagSets created by NamedFlagSets.
	NormalizeNameFunc func(f *pflag.FlagSet, name string) pflag.NormalizedName
}
```

## FlagSet
FlagSet是NamedFlagSets用于创建选项组的方法，选项组将会按照创建顺序(即调用FlagSet方法的顺序)进行排序
```go
// k8s.io/component-base/cli/flag/sectioned.go
// 传入要创建的选项组名
func (nfs *NamedFlagSets) FlagSet(name string) *pflag.FlagSet {
	if nfs.FlagSets == nil {
		nfs.FlagSets = map[string]*pflag.FlagSet{}
	}
	if _, ok := nfs.FlagSets[name]; !ok {
		flagSet := pflag.NewFlagSet(name, pflag.ExitOnError)
		flagSet.SetNormalizeFunc(pflag.CommandLine.GetNormalizeFunc())
		if nfs.NormalizeNameFunc != nil {
            // 如果指定了标准化选项名的函数，则覆盖cobra提供的函数
			flagSet.SetNormalizeFunc(nfs.NormalizeNameFunc)
		}
		nfs.FlagSets[name] = flagSet
		nfs.Order = append(nfs.Order, name)
	}
	return nfs.FlagSets[name]
}
```

## 打印帮助命令
```go
// k8s.io/component-base/cli/flag/sectioned.go
func SetUsageAndHelpFunc(cmd *cobra.Command, fss NamedFlagSets, cols int) {
    // 将cobra原生的帮助文档函数替换掉
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine())
		PrintSections(cmd.OutOrStderr(), fss, cols)
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine())
		PrintSections(cmd.OutOrStdout(), fss, cols)
	})
}

func PrintSections(w io.Writer, fss NamedFlagSets, cols int) {
    // 按顺序进行打印输出
	for _, name := range fss.Order {
		fs := fss.FlagSets[name]
		if !fs.HasFlags() {
			continue
		}

		wideFS := pflag.NewFlagSet("", pflag.ExitOnError)
		wideFS.AddFlagSet(fs)

		var zzz string
		if cols > 24 {
			zzz = strings.Repeat("z", cols-24)
			wideFS.Int(zzz, 0, strings.Repeat("z", cols-24))
		}

		var buf bytes.Buffer
		fmt.Fprintf(&buf, "\n%s flags:\n\n%s", strings.ToUpper(name[:1])+name[1:], wideFS.FlagUsagesWrapped(cols))

		if cols > 24 {
			i := strings.Index(buf.String(), zzz)
			lines := strings.Split(buf.String()[:i], "\n")
			fmt.Fprint(w, strings.Join(lines[:len(lines)-1], "\n"))
			fmt.Fprintln(w)
		} else {
			fmt.Fprint(w, buf.String())
		}
	}
}
```