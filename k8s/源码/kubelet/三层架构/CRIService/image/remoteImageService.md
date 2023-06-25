# remoteImageService
remoteImageService是ImageManagerService的实现类

## 结构体
```go
// pkg/kubelet/cri/remote/remote_image.go
type remoteImageService struct {
	timeout             time.Duration
	imageClient         runtimeapi.ImageServiceClient
	imageClientV1alpha2 runtimeapiV1alpha2.ImageServiceClient
}
```
1. imageClient：v1版本的grpc客户端
2. imageClientV1alpha2：v1alpha2版本的客户端

## 构造方法
```go
// pkg/kubelet/cri/remote/remote_image.go
func NewRemoteImageService(endpoint string, connectionTimeout time.Duration) (internalapi.ImageManagerService, error) {
	klog.V(3).InfoS("Connecting to image service", "endpoint", endpoint)
	addr, dialer, err := util.GetAddressAndDialer(endpoint)
	if err != nil {
		return nil, err
	}

	ctx, cancel := context.WithTimeout(context.Background(), connectionTimeout)
	defer cancel()

	conn, err := grpc.DialContext(ctx, addr, grpc.WithInsecure(), grpc.WithContextDialer(dialer), grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(maxMsgSize)))
	if err != nil {
		klog.ErrorS(err, "Connect remote image service failed", "address", addr)
		return nil, err
	}

	service := &remoteImageService{timeout: connectionTimeout}

    // 推断服务端所使用的grpc接口版本
	if err := service.determineAPIVersion(conn); err != nil {
		return nil, err
	}

	return service, nil

}

// 推断服务端所使用的grpc接口版本
func (r *remoteImageService) determineAPIVersion(conn *grpc.ClientConn) error {
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()

	klog.V(4).InfoS("Finding the CRI API image version")
    // 先默认创建v1版本客户端
	r.imageClient = runtimeapi.NewImageServiceClient(conn)

	if _, err := r.imageClient.ImageFsInfo(ctx, &runtimeapi.ImageFsInfoRequest{}); err == nil {
        klog.V(2).InfoS("Using CRI v1 image API")

	} else if status.Code(err) == codes.Unimplemented {
        // 如果ImageFsInfo未实现，则重新创建v1alpha版本的客户端
		klog.V(2).InfoS("Falling back to CRI v1alpha2 image API (deprecated)")
		r.imageClientV1alpha2 = runtimeapiV1alpha2.NewImageServiceClient(conn)

	} else {
		return fmt.Errorf("unable to determine image API version: %w", err)
	}

	return nil
}
```