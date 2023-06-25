# Run

## 命令行入口
```go
// cmd/kube-controller-manager/app/controllermanager.go
Run: func(cmd *cobra.Command, args []string) {
    // 处理--version
    verflag.PrintAndExitIfRequested()

    // Activate logging as soon as possible, after that
    // show flags with the final logging configuration.
    // 验证并启用日志参数
    if err := s.Logs.ValidateAndApply(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
    cliflag.PrintFlags(cmd.Flags())

    // 不启用非安全端口
    err := checkNonZeroInsecurePort(cmd.Flags())
    if err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }

    // 从options生成config
    c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
    if err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }

    // 完善config，开始运行服务
    if err := Run(c.Complete(), wait.NeverStop); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
},
```

## Run
```go
// cmd/kube-controller-manager/app/controllermanager.go
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())

    // 处理/configz路由
	if cfgz, err := configz.New(ConfigzName); err == nil {
		cfgz.Set(c.ComponentConfig)
	} else {
		klog.Errorf("unable to register configz: %v", err)
	}

	// Setup any healthz checks we will want to use.
	var checks []healthz.HealthChecker
	var electionChecker *leaderelection.HealthzAdaptor
	if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		electionChecker = leaderelection.NewLeaderHealthzAdaptor(time.Second * 20)
		checks = append(checks, electionChecker)
	}
	healthzHandler := controllerhealthz.NewMutableHealthzHandler(checks...)

	// Start the controller manager HTTP server
	// unsecuredMux is the handler for these controller *after* authn/authz filters have been applied
	var unsecuredMux *mux.PathRecorderMux
	if c.SecureServing != nil {
		// 生成非安全的分发器
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, healthzHandler)
		// 将非安全的分发器包装为安全的分发器
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		// TODO: handle stoppedCh returned by c.SecureServing.Serve
		// 开始服务
		if _, err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}

	clientBuilder, rootClientBuilder := createClientBuilders(c)

	saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController

	run := func(ctx context.Context, startSATokenController InitFunc, initializersFunc ControllerInitializersFunc) {

		// 构建Controller执行上下文环境
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			klog.Fatalf("error building controller context: %v", err)
		}
		// 执行ControllerInitializersFunc，获取所有Controller的InitFunc
		controllerInitializers := initializersFunc(controllerContext.LoopMode)
		if err := StartControllers(ctx, controllerContext, startSATokenController, controllerInitializers, unsecuredMux, healthzHandler); err != nil {
			klog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(stopCh)
		controllerContext.ObjectOrMetadataInformerFactory.Start(stopCh)
		close(controllerContext.InformersStarted)

		select {}
	}

	// No leader election, run directly
    // 在不需要leader选举时，直接运行服务
	// NewControllerInitializers返回保存着所有Controller运行函数的函数
	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		run(context.TODO(), saTokenControllerInitFunc, NewControllerInitializers)
		panic("unreachable")
	}

	// leader选举相关逻辑
	id, err := os.Hostname()
	if err != nil {
		return err
	}

	// add a uniquifier so that two processes on the same host don't accidentally both become active
	id = id + "_" + string(uuid.NewUUID())

	// leaderMigrator will be non-nil if and only if Leader Migration is enabled.
	var leaderMigrator *leadermigration.LeaderMigrator = nil

	// startSATokenController will be original saTokenControllerInitFunc if leader migration is not enabled.
	startSATokenController := saTokenControllerInitFunc

	// If leader migration is enabled, create the LeaderMigrator and prepare for migration
	if leadermigration.Enabled(&c.ComponentConfig.Generic) {
		klog.Infof("starting leader migration")

		leaderMigrator = leadermigration.NewLeaderMigrator(&c.ComponentConfig.Generic.LeaderMigration,
			"kube-controller-manager")

		// Wrap saTokenControllerInitFunc to signal readiness for migration after starting
		//  the controller.
		startSATokenController = func(ctx context.Context, controllerContext ControllerContext) (controller.Interface, bool, error) {
			defer close(leaderMigrator.MigrationReady)
			return saTokenControllerInitFunc(ctx, controllerContext)
		}
	}

	// Start the main lock
	go leaderElectAndRun(c, id, electionChecker,
		c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		c.ComponentConfig.Generic.LeaderElection.ResourceName,
		leaderelection.LeaderCallbacks{
			OnStartedLeading: func(ctx context.Context) {
				initializersFunc := NewControllerInitializers
				if leaderMigrator != nil {
					// If leader migration is enabled, we should start only non-migrated controllers
					//  for the main lock.
					initializersFunc = createInitializersFunc(leaderMigrator.FilterFunc, leadermigration.ControllerNonMigrated)
					klog.Info("leader migration: starting main controllers.")
				}
				run(ctx, startSATokenController, initializersFunc)
			},
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		})

	// If Leader Migration is enabled, proceed to attempt the migration lock.
	if leaderMigrator != nil {
		// Wait for Service Account Token Controller to start before acquiring the migration lock.
		// At this point, the main lock must have already been acquired, or the KCM process already exited.
		// We wait for the main lock before acquiring the migration lock to prevent the situation
		//  where KCM instance A holds the main lock while KCM instance B holds the migration lock.
		<-leaderMigrator.MigrationReady

		// Start the migration lock.
		go leaderElectAndRun(c, id, electionChecker,
			c.ComponentConfig.Generic.LeaderMigration.ResourceLock,
			c.ComponentConfig.Generic.LeaderMigration.LeaderName,
			leaderelection.LeaderCallbacks{
				OnStartedLeading: func(ctx context.Context) {
					klog.Info("leader migration: starting migrated controllers.")
					// DO NOT start saTokenController under migration lock
					run(ctx, nil, createInitializersFunc(leaderMigrator.FilterFunc, leadermigration.ControllerMigrated))
				},
				OnStoppedLeading: func() {
					klog.Fatalf("migration leaderelection lost")
				},
			})
	}

	select {}
}
```

## NewControllerInitializers
```go
// cmd/kube-controller-manager/app/controllermanager.go
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}
	controllers["endpoint"] = startEndpointController
	controllers["endpointslice"] = startEndpointSliceController
	controllers["endpointslicemirroring"] = startEndpointSliceMirroringController
	controllers["replicationcontroller"] = startReplicationController
	controllers["podgc"] = startPodGCController
	controllers["resourcequota"] = startResourceQuotaController
	controllers["namespace"] = startNamespaceController
	controllers["serviceaccount"] = startServiceAccountController
	controllers["garbagecollector"] = startGarbageCollectorController
	controllers["daemonset"] = startDaemonSetController
	controllers["job"] = startJobController
	controllers["deployment"] = startDeploymentController
	controllers["replicaset"] = startReplicaSetController
	controllers["horizontalpodautoscaling"] = startHPAController
	controllers["disruption"] = startDisruptionController
	controllers["statefulset"] = startStatefulSetController
	controllers["cronjob"] = startCronJobController
	controllers["csrsigning"] = startCSRSigningController
	controllers["csrapproving"] = startCSRApprovingController
	controllers["csrcleaner"] = startCSRCleanerController
	controllers["ttl"] = startTTLController
	controllers["bootstrapsigner"] = startBootstrapSignerController
	controllers["tokencleaner"] = startTokenCleanerController
	controllers["nodeipam"] = startNodeIpamController
	controllers["nodelifecycle"] = startNodeLifecycleController
	if loopMode == IncludeCloudLoops {
		controllers["service"] = startServiceController
		controllers["route"] = startRouteController
		controllers["cloud-node-lifecycle"] = startCloudNodeLifecycleController
		// TODO: volume controller into the IncludeCloudLoops only set.
	}
	controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
	controllers["attachdetach"] = startAttachDetachController
	controllers["persistentvolume-expander"] = startVolumeExpandController
	controllers["clusterrole-aggregation"] = startClusterRoleAggregrationController
	controllers["pvc-protection"] = startPVCProtectionController
	controllers["pv-protection"] = startPVProtectionController
	controllers["ttl-after-finished"] = startTTLAfterFinishedController
	controllers["root-ca-cert-publisher"] = startRootCACertPublisher
	controllers["ephemeral-volume"] = startEphemeralVolumeController
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) {
		controllers["storage-version-gc"] = startStorageVersionGCController
	}

	return controllers
}
```