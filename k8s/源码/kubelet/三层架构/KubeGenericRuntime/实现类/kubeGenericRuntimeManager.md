# kubeGenericRuntimeManager
KubeGenericRuntime接口的实现类，kubelet管理CRI的结构体

## 结构体
```go
// kubelet/kuberuntime/kuberuntime_manager.go
type kubeGenericRuntimeManager struct {
	runtimeName string
	recorder    record.EventRecorder
	osInterface kubecontainer.OSInterface

	// machineInfo contains the machine information.
	machineInfo *cadvisorapi.MachineInfo

	// Container GC manager
	containerGC *containerGC

	// Keyring for pulling images
	keyring credentialprovider.DockerKeyring

	// Runner of lifecycle events.
	runner kubecontainer.HandlerRunner

	// RuntimeHelper that wraps kubelet to generate runtime container options.
	runtimeHelper kubecontainer.RuntimeHelper

	// Health check results.
	livenessManager  proberesults.Manager
	readinessManager proberesults.Manager
	startupManager   proberesults.Manager

	// If true, enforce container cpu limits with CFS quota support
	cpuCFSQuota bool

	// CPUCFSQuotaPeriod sets the CPU CFS quota period value, cpu.cfs_period_us, defaults to 100ms
	cpuCFSQuotaPeriod metav1.Duration

	// wrapped image puller.
	imagePuller images.ImageManager

	// gRPC service clients
	runtimeService internalapi.RuntimeService
	imageService   internalapi.ImageManagerService

	// The version cache of runtime daemon.
	versionCache *cache.ObjectCache

	// The directory path for seccomp profiles.
	seccompProfileRoot string

	// Internal lifecycle event handlers for container resource management.
	internalLifecycle cm.InternalContainerLifecycle

	// A shim to legacy functions for backward compatibility.
	legacyLogProvider LegacyLogProvider

	// Manage container logs.
	logManager logs.ContainerLogManager

	// Manage RuntimeClass resources.
	runtimeClassManager *runtimeclass.Manager

	// Cache last per-container error message to reduce log spam
	logReduction *logreduction.LogReduction

	// PodState provider instance
	podStateProvider podStateProvider

	// Use RuntimeDefault as the default seccomp profile for all workloads.
	seccompDefault bool

	// MemorySwapBehavior defines how swap is used
	memorySwapBehavior string

	//Function to get node allocatable resources
	getNodeAllocatable func() v1.ResourceList

	// Memory throttling factor for MemoryQoS
	memoryThrottlingFactor float64
}
```

## 构造函数
```go
// pkg/kubelet/kuberuntime/kuberuntime_manager.go
func NewKubeGenericRuntimeManager(
	recorder record.EventRecorder,
	livenessManager proberesults.Manager,
	readinessManager proberesults.Manager,
	startupManager proberesults.Manager,
	rootDirectory string,
	machineInfo *cadvisorapi.MachineInfo,
	podStateProvider podStateProvider,
	osInterface kubecontainer.OSInterface,
	runtimeHelper kubecontainer.RuntimeHelper,
	httpClient types.HTTPGetter,
	imageBackOff *flowcontrol.Backoff,
	serializeImagePulls bool,
	imagePullQPS float32,
	imagePullBurst int,
	imageCredentialProviderConfigFile string,
	imageCredentialProviderBinDir string,
	cpuCFSQuota bool,
	cpuCFSQuotaPeriod metav1.Duration,
	runtimeService internalapi.RuntimeService,
	imageService internalapi.ImageManagerService,
	internalLifecycle cm.InternalContainerLifecycle,
	legacyLogProvider LegacyLogProvider,
	logManager logs.ContainerLogManager,
	runtimeClassManager *runtimeclass.Manager,
	seccompDefault bool,
	memorySwapBehavior string,
	getNodeAllocatable func() v1.ResourceList,
	memoryThrottlingFactor float64,
) (KubeGenericRuntime, error) {
	// New对象
	kubeRuntimeManager := &kubeGenericRuntimeManager{
		recorder:               recorder,
		cpuCFSQuota:            cpuCFSQuota,
		cpuCFSQuotaPeriod:      cpuCFSQuotaPeriod,
		seccompProfileRoot:     filepath.Join(rootDirectory, "seccomp"),
		livenessManager:        livenessManager,
		readinessManager:       readinessManager,
		startupManager:         startupManager,
		machineInfo:            machineInfo,
		osInterface:            osInterface,
		runtimeHelper:          runtimeHelper,
		runtimeService:         newInstrumentedRuntimeService(runtimeService),
		imageService:           newInstrumentedImageManagerService(imageService),
		internalLifecycle:      internalLifecycle,
		legacyLogProvider:      legacyLogProvider,
		logManager:             logManager,
		runtimeClassManager:    runtimeClassManager,
		logReduction:           logreduction.NewLogReduction(identicalErrorDelay),
		seccompDefault:         seccompDefault,
		memorySwapBehavior:     memorySwapBehavior,
		getNodeAllocatable:     getNodeAllocatable,
		memoryThrottlingFactor: memoryThrottlingFactor,
	}

	typedVersion, err := kubeRuntimeManager.getTypedVersion()
	if err != nil {
		klog.ErrorS(err, "Get runtime version failed")
		return nil, err
	}

	// Only matching kubeRuntimeAPIVersion is supported now
	// TODO: Runtime API machinery is under discussion at https://github.com/kubernetes/kubernetes/issues/28642
	if typedVersion.Version != kubeRuntimeAPIVersion {
		klog.ErrorS(err, "This runtime api version is not supported",
			"apiVersion", typedVersion.Version,
			"supportedAPIVersion", kubeRuntimeAPIVersion)
		return nil, ErrVersionNotSupported
	}

	kubeRuntimeManager.runtimeName = typedVersion.RuntimeName
	klog.InfoS("Container runtime initialized",
		"containerRuntime", typedVersion.RuntimeName,
		"version", typedVersion.RuntimeVersion,
		"apiVersion", typedVersion.RuntimeApiVersion)

	// If the container logs directory does not exist, create it.
	// TODO: create podLogsRootDirectory at kubelet.go when kubelet is refactored to
	// new runtime interface
	if _, err := osInterface.Stat(podLogsRootDirectory); os.IsNotExist(err) {
		if err := osInterface.MkdirAll(podLogsRootDirectory, 0755); err != nil {
			klog.ErrorS(err, "Failed to create pod log directory", "path", podLogsRootDirectory)
		}
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.KubeletCredentialProviders) && (imageCredentialProviderConfigFile != "" || imageCredentialProviderBinDir != "") {
		klog.InfoS("Flags --image-credential-provider-config or --image-credential-provider-bin-dir were set but the feature gate was disabled, these flags will be ignored",
			"featureGate", features.KubeletCredentialProviders)
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.KubeletCredentialProviders) && (imageCredentialProviderConfigFile != "" || imageCredentialProviderBinDir != "") {
		if err := plugin.RegisterCredentialProviderPlugins(imageCredentialProviderConfigFile, imageCredentialProviderBinDir); err != nil {
			klog.ErrorS(err, "Failed to register CRI auth plugins")
			os.Exit(1)
		}
	}
	kubeRuntimeManager.keyring = credentialprovider.NewDockerKeyring()

	kubeRuntimeManager.imagePuller = images.NewImageManager(
		kubecontainer.FilterEventRecorder(recorder),
		kubeRuntimeManager,
		imageBackOff,
		serializeImagePulls,
		imagePullQPS,
		imagePullBurst)
	kubeRuntimeManager.runner = lifecycle.NewHandlerRunner(httpClient, kubeRuntimeManager, kubeRuntimeManager)
	kubeRuntimeManager.containerGC = newContainerGC(runtimeService, podStateProvider, kubeRuntimeManager)
	kubeRuntimeManager.podStateProvider = podStateProvider

	kubeRuntimeManager.versionCache = cache.NewObjectCache(
		func() (interface{}, error) {
			return kubeRuntimeManager.getTypedVersion()
		},
		versionCacheTTL,
	)

	return kubeRuntimeManager, nil
}
```
