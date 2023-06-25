# leaseController
leaseController是k8s提供的用于管理Lease资源对象的Controller

## 结构体
```go
// k8s.io/component-helpers/apimachinery/lease/controller.go
type controller struct {
	client                     clientset.Interface
	leaseClient                coordclientset.LeaseInterface
	holderIdentity             string
	leaseNamespace             string
	leaseDurationSeconds       int32
	renewInterval              time.Duration
	clock                      clock.Clock
	onRepeatedHeartbeatFailure func()

	// latestLease is the latest lease which the controller updated or created
	latestLease *coordinationv1.Lease

	// newLeasePostProcessFunc allows customizing a lease object (e.g. setting OwnerReference)
	// before every time the lease is created/refreshed(updated). Note that an error will block
	// a lease CREATE, causing the controller to retry next time, but an error won't block a
	// lease UPDATE.
	newLeasePostProcessFunc ProcessLeaseFunc
}

type ProcessLeaseFunc func(*coordinationv1.Lease) error
```
1. holderIdentity：需要持有的Lease名
2. leaseNamespace：资源对象的命名空间，一般为"kube-node-lease"
3. leaseDurationSeconds：租约持续时间，默认40s
4. renewInterval：续约间隔，默认10s
5. onRepeatedHeartbeatFailure：续约操作失败时执行的函数
6. newLeasePostProcessFunc：修改Lease对象的钩子

## 构造函数
```go
// k8s.io/component-helpers/apimachinery/lease/controller.go
// 传入的leaseNamespace一般为"kube-node-lease"
func NewController(clock clock.Clock, client clientset.Interface, holderIdentity string, leaseDurationSeconds int32, onRepeatedHeartbeatFailure func(), renewInterval time.Duration, leaseNamespace string, newLeasePostProcessFunc ProcessLeaseFunc) Controller {
	var leaseClient coordclientset.LeaseInterface
	if client != nil {
        // 获取lease客户端
		leaseClient = client.CoordinationV1().Leases(leaseNamespace)
	}
	return &controller{
		client:                     client,
		leaseClient:                leaseClient,
		holderIdentity:             holderIdentity,
		leaseNamespace:             leaseNamespace,
		leaseDurationSeconds:       leaseDurationSeconds,
		renewInterval:              renewInterval,
		clock:                      clock,
		onRepeatedHeartbeatFailure: onRepeatedHeartbeatFailure,
		newLeasePostProcessFunc:    newLeasePostProcessFunc,
	}
}
```

## Run
主要循环
```go
func (c *controller) Run(stopCh <-chan struct{}) {
	if c.leaseClient == nil {
		klog.Infof("lease controller has nil lease client, will not claim or renew leases")
		return
	}
    // 循环执行c.sync，间隔时间c.renewInterval，抖动0.04，在函数执行完成后再开始计算时间
	wait.JitterUntil(c.sync, c.renewInterval, 0.04, true, stopCh)
}

func (c *controller) sync() {
	if c.latestLease != nil {
		// As long as the lease is not (or very rarely) updated by any other agent than the component itself,
		// we can optimistically assume it didn't change since our last update and try updating
		// based on the version from that time. Thanks to it we avoid GET call and reduce load
		// on etcd and kube-apiserver.
		// If at some point other agents will also be frequently updating the Lease object, this
		// can result in performance degradation, because we will end up with calling additional
		// GET/PUT - at this point this whole "if" should be removed.
        // 更新Lease
		err := c.retryUpdateLease(c.latestLease)
		if err == nil {
			return
		}
		klog.Infof("failed to update lease using latest lease, fallback to ensure lease, err: %v", err)
	}

    // 第一次，创建Lease
	lease, created := c.backoffEnsureLease()
	c.latestLease = lease
	// we don't need to update the lease if we just created it
    // 如果不是创建出来的Lease，则需要更新它
	if !created && lease != nil {
		if err := c.retryUpdateLease(lease); err != nil {
			klog.Errorf("%v, will retry after %v", err, c.renewInterval)
		}
	}
}
```

## backoffEnsureLease
如果Lease不存在，则创建；如果Lease现存，返回现存Lease。如果出错，成倍增长重试时间
```go
// k8s.io/component-helpers/apimachinery/lease/controller.go
func (c *controller) backoffEnsureLease() (*coordinationv1.Lease, bool) {
	var (
		lease   *coordinationv1.Lease
		created bool
		err     error
	)
	sleep := 100 * time.Millisecond
	for {
		lease, created, err = c.ensureLease()
		if err == nil {
			break
		}
        // 翻倍等待
		sleep = minDuration(2*sleep, maxBackoff)
		klog.Errorf("failed to ensure lease exists, will retry in %v, error: %v", sleep, err)
		// backoff wait
		c.clock.Sleep(sleep)
	}
	return lease, created
}

// 没找到Lease则创建，否则返回Lease
func (c *controller) ensureLease() (*coordinationv1.Lease, bool, error) {
	lease, err := c.leaseClient.Get(context.TODO(), c.holderIdentity, metav1.GetOptions{})
	if apierrors.IsNotFound(err) {
		// lease does not exist, create it.
		leaseToCreate, err := c.newLease(nil)
		// An error occurred during allocating the new lease (likely from newLeasePostProcessFunc).
		// Given that we weren't able to set the lease correctly, we simply
		// not create it this time - we will retry in the next iteration.
		if err != nil {
			return nil, false, nil
		}
		lease, err := c.leaseClient.Create(context.TODO(), leaseToCreate, metav1.CreateOptions{})
		if err != nil {
			return nil, false, err
		}
		return lease, true, nil
	} else if err != nil {
		// unexpected error getting lease
		return nil, false, err
	}
	// lease already existed
	return lease, false, nil
}
```

## 
更新Lease
```go
// k8s.io/component-apimachinery/lease/controller.go
// retryUpdateLease attempts to update the lease for maxUpdateRetries,
// call this once you're sure the lease has been created
func (c *controller) retryUpdateLease(base *coordinationv1.Lease) error {
	for i := 0; i < maxUpdateRetries; i++ {
		leaseToUpdate, _ := c.newLease(base)
		lease, err := c.leaseClient.Update(context.TODO(), leaseToUpdate, metav1.UpdateOptions{})
		if err == nil {
			c.latestLease = lease
			return nil
		}
		klog.Errorf("failed to update lease, error: %v", err)
		// OptimisticLockError requires getting the newer version of lease to proceed.
		if apierrors.IsConflict(err) {
			base, _ = c.backoffEnsureLease()
			continue
		}
		if i > 0 && c.onRepeatedHeartbeatFailure != nil {
			c.onRepeatedHeartbeatFailure()
		}
	}
	return fmt.Errorf("failed %d attempts to update lease", maxUpdateRetries)
}

func (c *controller) newLease(base *coordinationv1.Lease) (*coordinationv1.Lease, error) {
	// Use the bare minimum set of fields; other fields exist for debugging/legacy,
	// but we don't need to make component heartbeats more complicated by using them.
	var lease *coordinationv1.Lease
	if base == nil {
		lease = &coordinationv1.Lease{
			ObjectMeta: metav1.ObjectMeta{
				Name:      c.holderIdentity,
				Namespace: c.leaseNamespace,
			},
			Spec: coordinationv1.LeaseSpec{
				HolderIdentity:       pointer.StringPtr(c.holderIdentity),
				LeaseDurationSeconds: pointer.Int32Ptr(c.leaseDurationSeconds),
			},
		}
	} else {
		lease = base.DeepCopy()
	}
    // 更新RenewTime
	lease.Spec.RenewTime = &metav1.MicroTime{Time: c.clock.Now()}

	if c.newLeasePostProcessFunc != nil {
		err := c.newLeasePostProcessFunc(lease)
		return lease, err
	}

	return lease, nil
}
```