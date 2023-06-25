# syncNodeStatus
定时更新节点的状态到节点对应的Node资源对象

## syncNodeStatus
```go
// pkg/kubelet/kubelet_node_status.go
func (kl *Kubelet) syncNodeStatus() {
	kl.syncNodeStatusMux.Lock()
	defer kl.syncNodeStatusMux.Unlock()

	if kl.kubeClient == nil || kl.heartbeatClient == nil {
		return
	}

    // 判断是否需要kubelet创建自己所属的Node资源对象
	if kl.registerNode {
		// This will exit immediately if it doesn't need to do anything.
        // 如果已有资源对象，该方法会立即返回
		kl.registerWithAPIServer()
	}
    // 更新一次nodeStatus
	if err := kl.updateNodeStatus(); err != nil {
		klog.ErrorS(err, "Unable to update node status")
	}
}
```

## registerWithAPIServer
创建Kubelet对应的Node资源对象
```go
// pkg/kubelet/kubelet_node_status.go
func (kl *Kubelet) registerWithAPIServer() {
    // 注意这个字段在操作时未加锁
	if kl.registrationCompleted {
		return
	}
	step := 100 * time.Millisecond

	for {
        // 翻倍重试时间，0.1s 到 7s
		time.Sleep(step)
		step = step * 2
		if step >= 7*time.Second {
			step = 7 * time.Second
		}

        // 创建Node资源对象
		node, err := kl.initialNode(context.TODO())
		if err != nil {
			klog.ErrorS(err, "Unable to construct v1.Node object for kubelet")
			continue
		}

		klog.InfoS("Attempting to register node", "node", klog.KObj(node))
		registered := kl.tryRegisterWithAPIServer(node)
		if registered {
			klog.InfoS("Successfully registered node", "node", klog.KObj(node))
			kl.registrationCompleted = true
			return
		}
	}
}

func (kl *Kubelet) initialNode(ctx context.Context) (*v1.Node, error) {
	node := &v1.Node{
		ObjectMeta: metav1.ObjectMeta{
			Name: string(kl.nodeName),
			Labels: map[string]string{
				v1.LabelHostname:      kl.hostname,
				v1.LabelOSStable:      goruntime.GOOS,
				v1.LabelArchStable:    goruntime.GOARCH,
				kubeletapis.LabelOS:   goruntime.GOOS,
				kubeletapis.LabelArch: goruntime.GOARCH,
			},
		},
		Spec: v1.NodeSpec{
            // 这个字段已经过时，使用registerWithTaints代替
			Unschedulable: !kl.registerSchedulable,
		},
	}
	osLabels, err := getOSSpecificLabels()
	if err != nil {
		return nil, err
	}
	for label, value := range osLabels {
		node.Labels[label] = value
	}

	nodeTaints := make([]v1.Taint, len(kl.registerWithTaints))
	copy(nodeTaints, kl.registerWithTaints)
    // 默认添加一个不允许调度的Taint
	unschedulableTaint := v1.Taint{
		Key:    v1.TaintNodeUnschedulable,
		Effect: v1.TaintEffectNoSchedule,
	}

	// Taint node with TaintNodeUnschedulable when initializing
	// node to avoid race condition; refer to #63897 for more detail.
	if node.Spec.Unschedulable &&
		!taintutil.TaintExists(nodeTaints, &unschedulableTaint) {
		nodeTaints = append(nodeTaints, unschedulableTaint)
	}

	if kl.externalCloudProvider {
		taint := v1.Taint{
			Key:    cloudproviderapi.TaintExternalCloudProvider,
			Value:  "true",
			Effect: v1.TaintEffectNoSchedule,
		}

		nodeTaints = append(nodeTaints, taint)
	}
	if len(nodeTaints) > 0 {
		node.Spec.Taints = nodeTaints
	}
	// Initially, set NodeNetworkUnavailable to true.
	if kl.providerRequiresNetworkingConfiguration() {
		node.Status.Conditions = append(node.Status.Conditions, v1.NodeCondition{
			Type:               v1.NodeNetworkUnavailable,
			Status:             v1.ConditionTrue,
			Reason:             "NoRouteCreated",
			Message:            "Node created without a route",
			LastTransitionTime: metav1.NewTime(kl.clock.Now()),
		})
	}

	if kl.enableControllerAttachDetach {
		if node.Annotations == nil {
			node.Annotations = make(map[string]string)
		}

		klog.V(2).InfoS("Setting node annotation to enable volume controller attach/detach")
		node.Annotations[volutil.ControllerManagedAttachAnnotation] = "true"
	} else {
		klog.V(2).InfoS("Controller attach/detach is disabled for this node; Kubelet will attach and detach volumes")
	}

	if kl.keepTerminatedPodVolumes {
		if node.Annotations == nil {
			node.Annotations = make(map[string]string)
		}
		klog.V(2).InfoS("Setting node annotation to keep pod volumes of terminated pods attached to the node")
		node.Annotations[volutil.KeepTerminatedPodVolumesAnnotation] = "true"
	}

	// @question: should this be place after the call to the cloud provider? which also applies labels
	for k, v := range kl.nodeLabels {
		if cv, found := node.ObjectMeta.Labels[k]; found {
			klog.InfoS("the node label will overwrite default setting", "labelKey", k, "labelValue", v, "default", cv)
		}
		node.ObjectMeta.Labels[k] = v
	}

	if kl.providerID != "" {
		node.Spec.ProviderID = kl.providerID
	}

	if kl.cloud != nil {
		instances, ok := kl.cloud.Instances()
		if !ok {
			return nil, fmt.Errorf("failed to get instances from cloud provider")
		}

		// TODO: We can't assume that the node has credentials to talk to the
		// cloudprovider from arbitrary nodes. At most, we should talk to a
		// local metadata server here.
		var err error
		if node.Spec.ProviderID == "" {
			node.Spec.ProviderID, err = cloudprovider.GetInstanceProviderID(ctx, kl.cloud, kl.nodeName)
			if err != nil {
				return nil, err
			}
		}

		instanceType, err := instances.InstanceType(ctx, kl.nodeName)
		if err != nil {
			return nil, err
		}
		if instanceType != "" {
			klog.InfoS("Adding label from cloud provider", "labelKey", v1.LabelInstanceType, "labelValue", instanceType)
			node.ObjectMeta.Labels[v1.LabelInstanceType] = instanceType
			klog.InfoS("Adding node label from cloud provider", "labelKey", v1.LabelInstanceTypeStable, "labelValue", instanceType)
			node.ObjectMeta.Labels[v1.LabelInstanceTypeStable] = instanceType
		}
		// If the cloud has zone information, label the node with the zone information
		zones, ok := kl.cloud.Zones()
		if ok {
			zone, err := zones.GetZone(ctx)
			if err != nil {
				return nil, fmt.Errorf("failed to get zone from cloud provider: %v", err)
			}
			if zone.FailureDomain != "" {
				klog.InfoS("Adding node label from cloud provider", "labelKey", v1.LabelFailureDomainBetaZone, "labelValue", zone.FailureDomain)
				node.ObjectMeta.Labels[v1.LabelFailureDomainBetaZone] = zone.FailureDomain
				klog.InfoS("Adding node label from cloud provider", "labelKey", v1.LabelTopologyZone, "labelValue", zone.FailureDomain)
				node.ObjectMeta.Labels[v1.LabelTopologyZone] = zone.FailureDomain
			}
			if zone.Region != "" {
				klog.InfoS("Adding node label from cloud provider", "labelKey", v1.LabelFailureDomainBetaRegion, "labelValue", zone.Region)
				node.ObjectMeta.Labels[v1.LabelFailureDomainBetaRegion] = zone.Region
				klog.InfoS("Adding node label from cloud provider", "labelKey", v1.LabelTopologyRegion, "labelValue", zone.Region)
				node.ObjectMeta.Labels[v1.LabelTopologyRegion] = zone.Region
			}
		}
	}

    // 设置Node的状态
	kl.setNodeStatus(node)

	return node, nil
}
```

## updateNodeStatus
```go
// pkg/kubelet/kubelet_node_status.go
func (kl *Kubelet) updateNodeStatus() error {
	klog.V(5).InfoS("Updating node status")
	for i := 0; i < nodeStatusUpdateRetry; i++ {
		if err := kl.tryUpdateNodeStatus(i); err != nil {
			if i > 0 && kl.onRepeatedHeartbeatFailure != nil {
				kl.onRepeatedHeartbeatFailure()
			}
			klog.ErrorS(err, "Error updating node status, will retry")
		} else {
			return nil
		}
	}
	return fmt.Errorf("update node status exceeds retry count")
}

func (kl *Kubelet) tryUpdateNodeStatus(tryNumber int) error {
	// In large clusters, GET and PUT operations on Node objects coming
	// from here are the majority of load on apiserver and etcd.
	// To reduce the load on etcd, we are serving GET operations from
	// apiserver cache (the data might be slightly delayed but it doesn't
	// seem to cause more conflict - the delays are pretty small).
	// If it result in a conflict, all retries are served directly from etcd.
	opts := metav1.GetOptions{}
	if tryNumber == 0 {
		util.FromApiserverCache(&opts)
	}

    // 使用heartbeatClient来获取Node资源对象
	node, err := kl.heartbeatClient.CoreV1().Nodes().Get(context.TODO(), string(kl.nodeName), opts)
	if err != nil {
		return fmt.Errorf("error getting node %q: %v", kl.nodeName, err)
	}

	originalNode := node.DeepCopy()
	if originalNode == nil {
		return fmt.Errorf("nil %q node object", kl.nodeName)
	}

	podCIDRChanged := false
	if len(node.Spec.PodCIDRs) != 0 {
		// Pod CIDR could have been updated before, so we cannot rely on
		// node.Spec.PodCIDR being non-empty. We also need to know if pod CIDR is
		// actually changed.
		podCIDRs := strings.Join(node.Spec.PodCIDRs, ",")
		if podCIDRChanged, err = kl.updatePodCIDR(podCIDRs); err != nil {
			klog.ErrorS(err, "Error updating pod CIDR")
		}
	}

	areRequiredLabelsNotPresent := false
	osName, osLabelExists := node.Labels[v1.LabelOSStable]
	if !osLabelExists || osName != goruntime.GOOS {
		if len(node.Labels) == 0 {
			node.Labels = make(map[string]string)
		}
		node.Labels[v1.LabelOSStable] = goruntime.GOOS
		areRequiredLabelsNotPresent = true
	}
	// Set the arch if there is a mismatch
	arch, archLabelExists := node.Labels[v1.LabelArchStable]
	if !archLabelExists || arch != goruntime.GOARCH {
		if len(node.Labels) == 0 {
			node.Labels = make(map[string]string)
		}
		node.Labels[v1.LabelArchStable] = goruntime.GOARCH
		areRequiredLabelsNotPresent = true
	}

    // 更新Node资源对象
	kl.setNodeStatus(node)

    // 如果还没有到Update的时间
	now := kl.clock.Now()
	if now.Before(kl.lastStatusReportTime.Add(kl.nodeStatusReportFrequency)) {
        // 并且podCIDR没有改变，node的状态没有改变，也没有需要持久化的标签。则不请求api-server
		if !podCIDRChanged && !nodeStatusHasChanged(&originalNode.Status, &node.Status) && !areRequiredLabelsNotPresent {
			// We must mark the volumes as ReportedInUse in volume manager's dsw even
			// if no changes were made to the node status (no volumes were added or removed
			// from the VolumesInUse list).
			//
			// The reason is that on a kubelet restart, the volume manager's dsw is
			// repopulated and the volume ReportedInUse is initialized to false, while the
			// VolumesInUse list from the Node object still contains the state from the
			// previous kubelet instantiation.
			//
			// Once the volumes are added to the dsw, the ReportedInUse field needs to be
			// synced from the VolumesInUse list in the Node.Status.
			//
			// The MarkVolumesAsReportedInUse() call cannot be performed in dsw directly
			// because it does not have access to the Node object.
			// This also cannot be populated on node status manager init because the volume
			// may not have been added to dsw at that time.
			kl.volumeManager.MarkVolumesAsReportedInUse(node.Status.VolumesInUse)
			return nil
		}
	}

	// Patch the current status on the API server
	updatedNode, _, err := nodeutil.PatchNodeStatus(kl.heartbeatClient.CoreV1(), types.NodeName(kl.nodeName), originalNode, node)
	if err != nil {
		return err
	}
	kl.lastStatusReportTime = now
	kl.setLastObservedNodeAddresses(updatedNode.Status.Addresses)
	// If update finishes successfully, mark the volumeInUse as reportedInUse to indicate
	// those volumes are already updated in the node's status
	kl.volumeManager.MarkVolumesAsReportedInUse(updatedNode.Status.VolumesInUse)
	return nil
}
```