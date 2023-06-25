# kubeconfig
kubeconfig是k8s-client的客户端配置文件，可以由此配置文件生成restclient.Config  
k8s提供了`vendor/k8s.io/client-go/tools/clientcmd/client_config.ClientConfig`作为从kubeconfig生成restclient.Config的接口
1. DirectClientConfig：基础实现，实现了由kubeconfig生成restclient.Config的基础逻辑
2. DeferredLoadingClientConfig：延迟加载实现，在调用ClientConfig()方法时，才会加载

## 工具函数
```go
// k8s.io/client-go/tools/clientcmd/client_config.go
// 显示指定一个kubeconfig文件路径，由此生成restclient.Config，可以覆盖api-server的地址
func BuildConfigFromFlags(masterUrl, kubeconfigPath string) (*restclient.Config, error) {
	if kubeconfigPath == "" && masterUrl == "" {
		klog.Warning("Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.")
		kubeconfig, err := restclient.InClusterConfig()
		if err == nil {
			return kubeconfig, nil
		}
		klog.Warning("error creating inClusterConfig, falling back to default config: ", err)
	}
	return NewNonInteractiveDeferredLoadingClientConfig(
		&ClientConfigLoadingRules{ExplicitPath: kubeconfigPath},
		&ConfigOverrides{ClusterInfo: clientcmdapi.Cluster{Server: masterUrl}}).ClientConfig()
}

// k8s.io/client-go/tools/clientcmd/config.go
// 修改kubeconfig文件
func ModifyConfig(configAccess ConfigAccess, newConfig clientcmdapi.Config, relativizePaths bool) error {
	if UseModifyConfigLock {
		possibleSources := configAccess.GetLoadingPrecedence()
		// sort the possible kubeconfig files so we always "lock" in the same order
		// to avoid deadlock (note: this can fail w/ symlinks, but... come on).
		sort.Strings(possibleSources)
		for _, filename := range possibleSources {
			if err := lockFile(filename); err != nil {
				return err
			}
			defer unlockFile(filename)
		}
	}

	startingConfig, err := configAccess.GetStartingConfig()
	if err != nil {
		return err
	}

	// We need to find all differences, locate their original files, read a partial config to modify only that stanza and write out the file.
	// Special case the test for current context and preferences since those always write to the default file.
	if reflect.DeepEqual(*startingConfig, newConfig) {
		// nothing to do
		return nil
	}

	if startingConfig.CurrentContext != newConfig.CurrentContext {
		if err := writeCurrentContext(configAccess, newConfig.CurrentContext); err != nil {
			return err
		}
	}

	if !reflect.DeepEqual(startingConfig.Preferences, newConfig.Preferences) {
		if err := writePreferences(configAccess, newConfig.Preferences); err != nil {
			return err
		}
	}

	// Search every cluster, authInfo, and context.  First from new to old for differences, then from old to new for deletions
	for key, cluster := range newConfig.Clusters {
		startingCluster, exists := startingConfig.Clusters[key]
		if !reflect.DeepEqual(cluster, startingCluster) || !exists {
			destinationFile := cluster.LocationOfOrigin
			if len(destinationFile) == 0 {
				destinationFile = configAccess.GetDefaultFilename()
			}

			configToWrite, err := getConfigFromFile(destinationFile)
			if err != nil {
				return err
			}
			t := *cluster

			configToWrite.Clusters[key] = &t
			configToWrite.Clusters[key].LocationOfOrigin = destinationFile
			if relativizePaths {
				if err := RelativizeClusterLocalPaths(configToWrite.Clusters[key]); err != nil {
					return err
				}
			}

			if err := WriteToFile(*configToWrite, destinationFile); err != nil {
				return err
			}
		}
	}

	// seenConfigs stores a map of config source filenames to computed config objects
	seenConfigs := map[string]*clientcmdapi.Config{}

	for key, context := range newConfig.Contexts {
		startingContext, exists := startingConfig.Contexts[key]
		if !reflect.DeepEqual(context, startingContext) || !exists {
			destinationFile := context.LocationOfOrigin
			if len(destinationFile) == 0 {
				destinationFile = configAccess.GetDefaultFilename()
			}

			// we only obtain a fresh config object from its source file
			// if we have not seen it already - this prevents us from
			// reading and writing to the same number of files repeatedly
			// when multiple / all contexts share the same destination file.
			configToWrite, seen := seenConfigs[destinationFile]
			if !seen {
				var err error
				configToWrite, err = getConfigFromFile(destinationFile)
				if err != nil {
					return err
				}
				seenConfigs[destinationFile] = configToWrite
			}

			configToWrite.Contexts[key] = context
		}
	}

	// actually persist config object changes
	for destinationFile, configToWrite := range seenConfigs {
		if err := WriteToFile(*configToWrite, destinationFile); err != nil {
			return err
		}
	}

	for key, authInfo := range newConfig.AuthInfos {
		startingAuthInfo, exists := startingConfig.AuthInfos[key]
		if !reflect.DeepEqual(authInfo, startingAuthInfo) || !exists {
			destinationFile := authInfo.LocationOfOrigin
			if len(destinationFile) == 0 {
				destinationFile = configAccess.GetDefaultFilename()
			}

			configToWrite, err := getConfigFromFile(destinationFile)
			if err != nil {
				return err
			}
			t := *authInfo
			configToWrite.AuthInfos[key] = &t
			configToWrite.AuthInfos[key].LocationOfOrigin = destinationFile
			if relativizePaths {
				if err := RelativizeAuthInfoLocalPaths(configToWrite.AuthInfos[key]); err != nil {
					return err
				}
			}

			if err := WriteToFile(*configToWrite, destinationFile); err != nil {
				return err
			}
		}
	}

	for key, cluster := range startingConfig.Clusters {
		if _, exists := newConfig.Clusters[key]; !exists {
			destinationFile := cluster.LocationOfOrigin
			if len(destinationFile) == 0 {
				destinationFile = configAccess.GetDefaultFilename()
			}

			configToWrite, err := getConfigFromFile(destinationFile)
			if err != nil {
				return err
			}
			delete(configToWrite.Clusters, key)

			if err := WriteToFile(*configToWrite, destinationFile); err != nil {
				return err
			}
		}
	}

	for key, context := range startingConfig.Contexts {
		if _, exists := newConfig.Contexts[key]; !exists {
			destinationFile := context.LocationOfOrigin
			if len(destinationFile) == 0 {
				destinationFile = configAccess.GetDefaultFilename()
			}

			configToWrite, err := getConfigFromFile(destinationFile)
			if err != nil {
				return err
			}
			delete(configToWrite.Contexts, key)

			if err := WriteToFile(*configToWrite, destinationFile); err != nil {
				return err
			}
		}
	}

	for key, authInfo := range startingConfig.AuthInfos {
		if _, exists := newConfig.AuthInfos[key]; !exists {
			destinationFile := authInfo.LocationOfOrigin
			if len(destinationFile) == 0 {
				destinationFile = configAccess.GetDefaultFilename()
			}

			configToWrite, err := getConfigFromFile(destinationFile)
			if err != nil {
				return err
			}
			delete(configToWrite.AuthInfos, key)

			if err := WriteToFile(*configToWrite, destinationFile); err != nil {
				return err
			}
		}
	}

	return nil
}
```