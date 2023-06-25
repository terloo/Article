# remoteRuntimeService
remoteRuntimeservice是CRIService中RuntimeService的实现类，使用远程调用grpc的方式来进行容器运行时的操作

## 结构体
```go
// pkg/kubelet/cri/remote/remote_runtime.go
type remoteRuntimeService struct {
	timeout               time.Duration
	runtimeClient         runtimeapi.RuntimeServiceClient
	runtimeClientV1alpha2 runtimeapiV1alpha2.RuntimeServiceClient
	// Cache last per-container error message to reduce log spam
	logReduction *logreduction.LogReduction
}
```
1. timeout：grpc连接和调用超时时间
2. runtimeClient：grpc调用客户端，v1版本
3. runtimeClientV1alpha2：v1alpha2版本的客户端(过时)，在nil时使用v1版
4. logReduction：日志减少器，通过其ShouldMessageBePrinted(message string, parentID string)来减少同一个容器同一日志的出现次数

## 构造函数
```go
// pkg/kubelet/cri/remote/remote_runtime.go
func NewRemoteRuntimeService(endpoint string, connectionTimeout time.Duration) (internalapi.RuntimeService, error) {
	klog.V(3).InfoS("Connecting to runtime service", "endpoint", endpoint)
    // 解析地址和协议
	addr, dialer, err := util.GetAddressAndDialer(endpoint)
	if err != nil {
		return nil, err
	}
	ctx, cancel := context.WithTimeout(context.Background(), connectionTimeout)
	defer cancel()

    // grpc连接
	conn, err := grpc.DialContext(ctx, addr, grpc.WithInsecure(), grpc.WithContextDialer(dialer), grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(maxMsgSize)))
	if err != nil {
		klog.ErrorS(err, "Connect remote runtime failed", "address", addr)
		return nil, err
	}

	service := &remoteRuntimeService{
		timeout:      connectionTimeout,
		logReduction: logreduction.NewLogReduction(identicalErrorDelay),
	}

    // 推断服务端使用的grpc api版本
	if err := service.determineAPIVersion(conn); err != nil {
		return nil, err
	}

	return service, nil
}

func (r *remoteRuntimeService) determineAPIVersion(conn *grpc.ClientConn) error {
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()

	klog.V(4).InfoS("Finding the CRI API runtime version")
    // 创建一个客户端，先默认服务端接口版本是v1
	r.runtimeClient = runtimeapi.NewRuntimeServiceClient(conn)

    // 调用v1.Version接口获取Runtime版本，未报错的是v1
	if _, err := r.runtimeClient.Version(ctx, &runtimeapi.VersionRequest{}); err == nil {
		klog.V(2).InfoS("Using CRI v1 runtime API")

	} else if status.Code(err) == codes.Unimplemented {
        // 报错未实现的是v1alpha2
		klog.V(2).InfoS("Falling back to CRI v1alpha2 runtime API (deprecated)")
        // 新建v1alpha2版本客户端
		r.runtimeClientV1alpha2 = runtimeapiV1alpha2.NewRuntimeServiceClient(conn)

	} else {
		return fmt.Errorf("unable to determine runtime API version: %w", err)
	}

	return nil
}
```

## 创建pod沙箱环境
```go
// pkg/kubelet/cri/runtime/remote_runtime.go
func (r *remoteRuntimeService) RunPodSandbox(config *runtimeapi.PodSandboxConfig, runtimeHandler string) (string, error) {
	// Use 2 times longer timeout for sandbox operation (4 mins by default)
	// TODO: Make the pod sandbox timeout configurable.
    // 创建pod沙箱环境的超时时间是2倍指定时间
	timeout := r.timeout * 2

	klog.V(10).InfoS("[RemoteRuntimeService] RunPodSandbox", "config", config, "runtimeHandler", runtimeHandler, "timeout", timeout)

	ctx, cancel := getContextWithTimeout(timeout)
	defer cancel()

	var podSandboxID string
	if r.useV1API() {
		resp, err := r.runtimeClient.RunPodSandbox(ctx, &runtimeapi.RunPodSandboxRequest{
			Config:         config,
			RuntimeHandler: runtimeHandler,
		})

		if err != nil {
			klog.ErrorS(err, "RunPodSandbox from runtime service failed")
			return "", err
		}
		podSandboxID = resp.PodSandboxId
	} else {
		resp, err := r.runtimeClientV1alpha2.RunPodSandbox(ctx, &runtimeapiV1alpha2.RunPodSandboxRequest{
			Config:         v1alpha2PodSandboxConfig(config),
			RuntimeHandler: runtimeHandler,
		})

		if err != nil {
			klog.ErrorS(err, "RunPodSandbox from runtime service failed")
			return "", err
		}
		podSandboxID = resp.PodSandboxId
	}

	if podSandboxID == "" {
		errorMessage := fmt.Sprintf("PodSandboxId is not set for sandbox %q", config.Metadata)
		err := errors.New(errorMessage)
		klog.ErrorS(err, "RunPodSandbox failed")
		return "", err
	}

	klog.V(10).InfoS("[RemoteRuntimeService] RunPodSandbox Response", "podSandboxID", podSandboxID)

	return podSandboxID, nil
}
```

## 创建Container
Pod的沙盒环境创建完毕后，就可以使用podSandBoxID来创建Container
```go
// pkg/kubelet/cri/remote/remote_runtime.go
func (r *remoteRuntimeService) CreateContainer(podSandBoxID string, config *runtimeapi.ContainerConfig, sandboxConfig *runtimeapi.PodSandboxConfig) (string, error) {
	klog.V(10).InfoS("[RemoteRuntimeService] CreateContainer", "podSandboxID", podSandBoxID, "timeout", r.timeout)
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()

	if r.useV1API() {
		return r.createContainerV1(ctx, podSandBoxID, config, sandboxConfig)
	}

	return r.createContainerV1alpha2(ctx, podSandBoxID, config, sandboxConfig)
}
```