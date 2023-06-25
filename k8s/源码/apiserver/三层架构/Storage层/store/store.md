# store
store是真正与Etcd集群交互的资源存储对象，对Etcd的官方库进行了封装，实现了storage层接口。位于`k8s.io/pkg/apiserver/storage/etcd3`
**每一个GR(GK)都会实例化一个store**

## 结构体
```go
// vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go
type store struct {
	// etcd的client库操作对象
	client              *clientv3.Client
	codec               runtime.Codec
	versioner           storage.Versioner
	transformer         value.Transformer
	pathPrefix          string
	groupResource       schema.GroupResource
	groupResourceString string
	watcher             *watcher
	pagingEnabled       bool
	leaseManager        *leaseManager
}
```
1. pathPrefix：路径前缀 /api 或者/apis
2. groupResource：该store对应的GR


## 构造函数
```go
// vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go
// 传入clientv3.Client，对其进行封装
func New(c *clientv3.Client, codec runtime.Codec, newFunc func() runtime.Object, prefix string, groupResource schema.GroupResource, transformer value.Transformer, pagingEnabled bool, leaseManagerConfig LeaseManagerConfig) storage.Interface {
	return newStore(c, codec, newFunc, prefix, groupResource, transformer, pagingEnabled, leaseManagerConfig)
}

func newStore(c *clientv3.Client, codec runtime.Codec, newFunc func() runtime.Object, prefix string, groupResource schema.GroupResource, transformer value.Transformer, pagingEnabled bool, leaseManagerConfig LeaseManagerConfig) *store {
	versioner := APIObjectVersioner{}
	result := &store{
		client:        c,
		codec:         codec,
		versioner:     versioner,
		transformer:   transformer,
		pagingEnabled: pagingEnabled,
		// for compatibility with etcd2 impl.
		// no-op for default prefix of '/registry'.
		// keeps compatibility with etcd2 impl for custom prefixes that don't start with '/'
		pathPrefix:          path.Join("/", prefix),
		groupResource:       groupResource,
		groupResourceString: groupResource.String(),
		watcher:             newWatcher(c, codec, newFunc, versioner, transformer),
		leaseManager:        newDefaultLeaseManager(c, leaseManagerConfig),
	}
	return result
}
```

## 存储和获取对象基本流程
1. 存储过程：runtime.Encode编码 -> 存储数据 -> 更新资源对象的RV
2. 获取过程：获取数据 -> runtime.Decode解码 -> 更新资源对象的RV

## 增
```go
// vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go
func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
	if version, err := s.versioner.ObjectResourceVersion(obj); err == nil && version != 0 {
		return errors.New("resourceVersion should not be set on objects to be created")
	}
	// 将SelfLink和ResourceVersion改为空字符串
	if err := s.versioner.PrepareObjectForStorage(obj); err != nil {
		return fmt.Errorf("PrepareObjectForStorage failed: %v", err)
	}
	// 将资源对象编码成protobuf
	data, err := runtime.Encode(s.codec, obj)
	if err != nil {
		return err
	}
	// 把key拼接上前缀
	key = path.Join(s.pathPrefix, key)

	// 如果有ttl，获取etcd的TTL Option
	opts, err := s.ttlOpts(ctx, int64(ttl))
	if err != nil {
		return err
	}

	// 如果需要，对数据进行加密
	newData, err := s.transformer.TransformToStorage(data, authenticatedDataString(key))
	if err != nil {
		return storage.NewInternalError(err.Error())
	}

	startTime := time.Now()

	// 进行一次事务操作，防止并发添加多个资源对象
	txnResp, err := s.client.KV.Txn(ctx).If(
		notFound(key),
	).Then(
		clientv3.OpPut(key, string(newData), opts...),
	).Commit()
	// 记录Etcd请求延迟
	metrics.RecordEtcdRequestLatency("create", getTypeName(obj), startTime)
	if err != nil {
		return err
	}
	if !txnResp.Succeeded {
		return storage.NewKeyExistsError(key, 0)
	}

	if out != nil {
		putResp := txnResp.Responses[0].GetResponsePut()
		return decode(s.codec, s.versioner, data, out, putResp.Header.Revision)
	}
	return nil
}
```

## 查
```go
// vendor/k8s.io/apiserver/pkg/storage/etcd3/store.go
func (s *store) Get(ctx context.Context, key string, opts storage.GetOptions, out runtime.Object) error {
	key = path.Join(s.pathPrefix, key)
	startTime := time.Now()
	// 从etcdclient中获取
	getResp, err := s.client.KV.Get(ctx, key)
	metrics.RecordEtcdRequestLatency("get", getTypeName(out), startTime)
	if err != nil {
		return err
	}
	if err = s.validateMinimumResourceVersion(opts.ResourceVersion, uint64(getResp.Header.Revision)); err != nil {
		return err
	}

	if len(getResp.Kvs) == 0 {
		if opts.IgnoreNotFound {
			return runtime.SetZeroValue(out)
		}
		return storage.NewKeyNotFoundError(key, 0)
	}
	kv := getResp.Kvs[0]

	// 如果需要，解密
	data, _, err := s.transformer.TransformFromStorage(kv.Value, authenticatedDataString(key))
	if err != nil {
		return storage.NewInternalError(err.Error())
	}

	// 解码
	return decode(s.codec, s.versioner, data, out, kv.ModRevision)
}

func decode(codec runtime.Codec, versioner storage.Versioner, value []byte, objPtr runtime.Object, rev int64) error {
	if _, err := conversion.EnforcePtr(objPtr); err != nil {
		return fmt.Errorf("unable to convert output object to pointer: %v", err)
	}

	// 调用s.codec进行解码
	_, _, err := codec.Decode(value, nil, objPtr)
	if err != nil {
		return err
	}
	// being unable to set the version does not prevent the object from being extracted
	// 更新资源对象版本
	if err := versioner.UpdateObject(objPtr, uint64(rev)); err != nil {
		klog.Errorf("failed to update object version: %v", err)
	}
	return nil
}
```