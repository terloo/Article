# NewCommand

## NewCommand
```go
// cmd/kubelet/app/server.go
func NewKubeletCommand() *cobra.Command {
    // 创建命令行Flag集
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
    // kubelet的所有Flag
	kubeletFlags := options.NewKubeletFlags()
    // 配置类
	kubeletConfig, err := options.NewKubeletConfiguration()
	// programmer error
	if err != nil {
		klog.ErrorS(err, "Failed to create a new kubelet configuration")
		os.Exit(1)
	}

	cmd := &cobra.Command{/*省略*/}

	// keep cleanFlagSet separate, so Cobra doesn't pollute it with the global flags
    // 添加kubelet的Flags到cleanFlagSet
	kubeletFlags.AddFlags(cleanFlagSet)
    // 添加kubeletConfig中的字段到cleanFlagSet，由于这些字段推荐通过config文件进行解析，所以将其全部标志为过期
	options.AddKubeletConfigFlags(cleanFlagSet, kubeletConfig)
    // 添加GlobalFlag
	options.AddGlobalFlags(cleanFlagSet)
	cleanFlagSet.BoolP("help", "h", false, fmt.Sprintf("help for %s", cmd.Name()))

	// ugly, but necessary, because Cobra's default UsageFunc and HelpFunc pollute the flagset with global flags
    // 重写命令行help和usage信息的生成函数
	const usageFmt = "Usage:\n  %s\n\nFlags:\n%s"
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine(), cleanFlagSet.FlagUsagesWrapped(2))
	})

	return cmd
}
```

## cobra.Command
```go
cmd := &cobra.Command{
		Use: componentKubelet,
		Long: `The kubelet is the primary "node agent" that runs on each
node. It can register the node with the apiserver using one of: the hostname; a flag to
override the hostname; or specific logic for a cloud provider.

The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object
that describes a pod. The kubelet takes a set of PodSpecs that are provided through
various mechanisms (primarily through the apiserver) and ensures that the containers
described in those PodSpecs are running and healthy. The kubelet doesn't manage
containers which were not created by Kubernetes.

Other than from an PodSpec from the apiserver, there are three ways that a container
manifest can be provided to the Kubelet.

File: Path passed as a flag on the command line. Files under this path will be monitored
periodically for updates. The monitoring period is 20s by default and is configurable
via a flag.

HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint
is checked every 20 seconds (also configurable with a flag).

HTTP server: The kubelet can also listen for HTTP and respond to a simple API
(underspec'd currently) to submit a new manifest.`,
		// The Kubelet has special flag parsing requirements to enforce flag precedence rules,
		// so we do all our parsing manually in Run, below.
		// DisableFlagParsing=true provides the full set of flags passed to the kubelet in the
		// `args` arg to Run, without Cobra's interference.
		// 禁用cobra自动解析
		DisableFlagParsing: true,
		Run: func(cmd *cobra.Command, args []string) {
			// initial flag parse, since we disable cobra's flag parsing
			// 手动解析flag
			if err := cleanFlagSet.Parse(args); err != nil {
				klog.ErrorS(err, "Failed to parse kubelet flag")
				cmd.Usage()
				os.Exit(1)
			}

			// check if there are non-flag arguments in the command line
			cmds := cleanFlagSet.Args()
			if len(cmds) > 0 {
				klog.ErrorS(nil, "Unknown command", "command", cmds[0])
				cmd.Usage()
				os.Exit(1)
			}

			// short-circuit on help
			// --help选项
			help, err := cleanFlagSet.GetBool("help")
			if err != nil {
				klog.InfoS(`"help" flag is non-bool, programmer error, please correct`)
				os.Exit(1)
			}
			if help {
				cmd.Help()
				return
			}

			// short-circuit on verflag
			// --version选项
			verflag.PrintAndExitIfRequested()

			// set feature gates from initial flags-based config
			if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
				klog.ErrorS(err, "Failed to set feature gates from initial flags-based config")
				os.Exit(1)
			}

			// validate the initial KubeletFlags
			if err := options.ValidateKubeletFlags(kubeletFlags); err != nil {
				klog.ErrorS(err, "Failed to validate kubelet flags")
				os.Exit(1)
			}

			// 如果使用非docker的CRI，则不用指定pod沙盒环境镜像
			if kubeletFlags.ContainerRuntime == "remote" && cleanFlagSet.Changed("pod-infra-container-image") {
				klog.InfoS("Warning: For remote container runtime, --pod-infra-container-image is ignored in kubelet, which should be set in that remote runtime instead")
			}

			// load kubelet config file, if provided
			// 加载配置文件
			if configFile := kubeletFlags.KubeletConfigFile; len(configFile) > 0 {
				kubeletConfig, err = loadConfigFile(configFile)
				if err != nil {
					klog.ErrorS(err, "Failed to load kubelet config file", "path", configFile)
					os.Exit(1)
				}
				// We must enforce flag precedence by re-parsing the command line into the new object.
				// This is necessary to preserve backwards-compatibility across binary upgrades.
				// See issue #56171 for more details.
				// 用命令行中的flag配置覆盖配置文件中的配置
				if err := kubeletConfigFlagPrecedence(kubeletConfig, args); err != nil {
					klog.ErrorS(err, "Failed to precedence kubeletConfigFlag")
					os.Exit(1)
				}
				// update feature gates based on new config
				if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
					klog.ErrorS(err, "Failed to set feature gates from initial flags-based config")
					os.Exit(1)
				}
			}

			// We always validate the local configuration (command line + config file).
			// This is the default "last-known-good" config for dynamic config, and must always remain valid.
			// 验证配置文件配置
			if err := kubeletconfigvalidation.ValidateKubeletConfiguration(kubeletConfig); err != nil {
				klog.ErrorS(err, "Failed to validate kubelet configuration", "path", kubeletConfig)
				os.Exit(1)
			}

			if (kubeletConfig.KubeletCgroups != "" && kubeletConfig.KubeReservedCgroup != "") && (strings.Index(kubeletConfig.KubeletCgroups, kubeletConfig.KubeReservedCgroup) != 0) {
				klog.InfoS("unsupported configuration:KubeletCgroups is not within KubeReservedCgroup")
			}

			// use dynamic kubelet config, if enabled
			// 如果启动，使用动态kubeletconfig
			var kubeletConfigController *dynamickubeletconfig.Controller
			if dynamicConfigDir := kubeletFlags.DynamicConfigDir.Value(); len(dynamicConfigDir) > 0 {
				var dynamicKubeletConfig *kubeletconfiginternal.KubeletConfiguration
				dynamicKubeletConfig, kubeletConfigController, err = BootstrapKubeletConfigController(dynamicConfigDir,
					func(kc *kubeletconfiginternal.KubeletConfiguration) error {
						// Here, we enforce flag precedence inside the controller, prior to the controller's validation sequence,
						// so that we get a complete validation at the same point where we can decide to reject dynamic config.
						// This fixes the flag-precedence component of issue #63305.
						// See issue #56171 for general details on flag precedence.
						return kubeletConfigFlagPrecedence(kc, args)
					})
				if err != nil {
					klog.ErrorS(err, "Failed to bootstrap a configuration controller", "dynamicConfigDir", dynamicConfigDir)
					os.Exit(1)
				}
				// If we should just use our existing, local config, the controller will return a nil config
				if dynamicKubeletConfig != nil {
					kubeletConfig = dynamicKubeletConfig
					// Note: flag precedence was already enforced in the controller, prior to validation,
					// by our above transform function. Now we simply update feature gates from the new config.
					if err := utilfeature.DefaultMutableFeatureGate.SetFromMap(kubeletConfig.FeatureGates); err != nil {
						klog.ErrorS(err, "Failed to set feature gates from initial flags-based config")
						os.Exit(1)
					}
				}
			}

			// Config and flags parsed, now we can initialize logging.
			// 初始化日志
			logs.InitLogs()
			logOption := &logs.Options{Config: kubeletConfig.Logging}
			if err := logOption.ValidateAndApply(); err != nil {
				klog.ErrorS(err, "Failed to initialize logging")
				os.Exit(1)
			}
			cliflag.PrintFlags(cleanFlagSet)

			// construct a KubeletServer from kubeletFlags and kubeletConfig
			// 构建kubeletServer
			kubeletServer := &options.KubeletServer{
				// 命令行Flag
				KubeletFlags:         *kubeletFlags,
				// 配置类
				KubeletConfiguration: *kubeletConfig,
			}

			// use kubeletServer to construct the default KubeletDeps
			// 使用kubeletServer构建默认的KubeletDeps
			kubeletDeps, err := UnsecuredDependencies(kubeletServer, utilfeature.DefaultFeatureGate)
			if err != nil {
				klog.ErrorS(err, "Failed to construct kubelet dependencies")
				os.Exit(1)
			}

			// add the kubelet config controller to kubeletDeps
			// 将动态kubelet配置文件控制器添加到kubeletDeps中
			kubeletDeps.KubeletConfigController = kubeletConfigController

			if err := checkPermissions(); err != nil {
				klog.ErrorS(err, "kubelet running with insufficient permissions")
			}
			// set up signal context here in order to be reused by kubelet and docker shim
			ctx := genericapiserver.SetupSignalContext()

			// 配置信息脱敏并打印
			// make the kubelet's config safe for logging
			config := kubeletServer.KubeletConfiguration.DeepCopy()
			for k := range config.StaticPodURLHeader {
				config.StaticPodURLHeader[k] = []string{"<masked>"}
			}
			// log the kubelet's config for inspection
			klog.V(5).InfoS("KubeletConfiguration", "configuration", config)

			// run the kubelet
			// 启动kubelet
			if err := Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate); err != nil {
				klog.ErrorS(err, "Failed to run kubelet")
				os.Exit(1)
			}
		},
	}
```