# podStorage
podStorage接收所有来源channel中的PodUpdate操作，**与本地缓存数据整合后**，将整合后的处理操作包装为PodUpdate传入updates channel中  
核心方法checkAndUpdatePod：比较新旧Pod，判断是否为有意义的更改，是否为优雅删除，是否为状态更新

## 接口
```go
// pkg/util/config/config.go
type Merger interface {
	// Invoked when a change from a source is received.  May also function as an incremental
	// merger if you wish to consume changes incrementally.  Must be reentrant when more than
	// one source is defined.
	Merge(source string, update interface{}) error
}
```

## 实现类
```go
// pkg/kubelet/config/config.go
type podStorage struct {
	podLock sync.RWMutex
	// map of source name to pod uid to pod reference
	pods map[string]map[types.UID]*v1.Pod
	mode PodConfigNotificationMode

	// ensures that updates are delivered in strict order
	// on the updates channel
	updateLock sync.Mutex
	updates    chan<- kubetypes.PodUpdate

	// contains the set of all sources that have sent at least one SET
	sourcesSeenLock sync.RWMutex
	sourcesSeen     sets.String

	// the EventRecorder to use
	recorder record.EventRecorder
}
```
1. podLock：更新本地pods缓存时使用的锁
2. pods：保存了所有Pod的所有来源传入的Pod的引用，来源名字 -> pod的Uid -> pod引用
3. mode：该PodConfig的提醒模式
4. updateLock：保持update channel有序使用的锁
5. updates：更新渠道
6. sourceSeen：保存至少发送过一次消息的来源的名字
7. recorder：事件记录器，newPods如果name有重复，将会记录该事件

## PodConfigNotificationMode
```go
// pkg/kubelet/config/config.go
type PodConfigNotificationMode int

const (
	// PodConfigNotificationUnknown is the default value for
	// PodConfigNotificationMode when uninitialized.
	PodConfigNotificationUnknown PodConfigNotificationMode = iota
	// PodConfigNotificationSnapshot delivers the full configuration as a SET whenever
	// any change occurs.
	PodConfigNotificationSnapshot
	// PodConfigNotificationSnapshotAndUpdates delivers an UPDATE and DELETE message whenever pods are
	// changed, and a SET message if there are any additions or removals.
	PodConfigNotificationSnapshotAndUpdates
	// PodConfigNotificationIncremental delivers ADD, UPDATE, DELETE, REMOVE, RECONCILE to the update channel.
	PodConfigNotificationIncremental
)
```

## 构造函数
```go
func newPodStorage(updates chan<- kubetypes.PodUpdate, mode PodConfigNotificationMode, recorder record.EventRecorder) *podStorage {
	return &podStorage{
		pods:        make(map[string]map[types.UID]*v1.Pod),
		mode:        mode,
		updates:     updates,
		sourcesSeen: sets.String{},
		recorder:    recorder,
	}
}
```

## Merge
实现了Merger接口
```go
// pkg/kubelet/config/config.go
func (s *podStorage) Merge(source string, change interface{}) error {
    // 加更新锁，保持有序
	s.updateLock.Lock()
	defer s.updateLock.Unlock()

	seenBefore := s.sourcesSeen.Has(source)
	// 整合本地缓存使用的方法
	adds, updates, deletes, removes, reconciles := s.merge(source, change)
	firstSet := !seenBefore && s.sourcesSeen.Has(source)

	// deliver update notifications
	switch s.mode {
	case PodConfigNotificationIncremental:
		if len(removes.Pods) > 0 {
			s.updates <- *removes
		}
		if len(adds.Pods) > 0 {
			s.updates <- *adds
		}
		if len(updates.Pods) > 0 {
			s.updates <- *updates
		}
		if len(deletes.Pods) > 0 {
			s.updates <- *deletes
		}
		if firstSet && len(adds.Pods) == 0 && len(updates.Pods) == 0 && len(deletes.Pods) == 0 {
			// Send an empty update when first seeing the source and there are
			// no ADD or UPDATE or DELETE pods from the source. This signals kubelet that
			// the source is ready.
			s.updates <- *adds
		}
		// Only add reconcile support here, because kubelet doesn't support Snapshot update now.
		if len(reconciles.Pods) > 0 {
			s.updates <- *reconciles
		}

	case PodConfigNotificationSnapshotAndUpdates:
		if len(removes.Pods) > 0 || len(adds.Pods) > 0 || firstSet {
			s.updates <- kubetypes.PodUpdate{Pods: s.MergedState().([]*v1.Pod), Op: kubetypes.SET, Source: source}
		}
		if len(updates.Pods) > 0 {
			s.updates <- *updates
		}
		if len(deletes.Pods) > 0 {
			s.updates <- *deletes
		}

	case PodConfigNotificationSnapshot:
		if len(updates.Pods) > 0 || len(deletes.Pods) > 0 || len(adds.Pods) > 0 || len(removes.Pods) > 0 || firstSet {
			s.updates <- kubetypes.PodUpdate{Pods: s.MergedState().([]*v1.Pod), Op: kubetypes.SET, Source: source}
		}

	case PodConfigNotificationUnknown:
		fallthrough
	default:
		panic(fmt.Sprintf("unsupported PodConfigNotificationMode: %#v", s.mode))
	}

	return nil
}
```

## merge
将来源传入的PodUpdate与本地缓存信息向比较，生成新的PodUpdate
```go
// pkg/kubelet/config/config.go
func (s *podStorage) merge(source string, change interface{}) (adds, updates, deletes, removes, reconciles *kubetypes.PodUpdate) {
    // 加Pod锁
	s.podLock.Lock()
	defer s.podLock.Unlock()

	addPods := []*v1.Pod{}
	updatePods := []*v1.Pod{}
	deletePods := []*v1.Pod{}
	removePods := []*v1.Pod{}
	reconcilePods := []*v1.Pod{}

    // 先获取该来源所有Pod
	pods := s.pods[source]
	if pods == nil {
		pods = make(map[types.UID]*v1.Pod)
	}

	// updatePodFunc is the local function which updates the pod cache *oldPods* with new pods *newPods*.
	// After updated, new pod will be stored in the pod cache *pods*.
	// Notice that *pods* and *oldPods* could be the same cache.
    // newPods：新Pod，推送过来的数据
    // oldPods：缓存Pod，podStorage.Pods中保存的该source的所有数据
    // pods：合并结果的保存处，oldPods和pods可能指向同一个对象
    // 该方法是将newPods中所有的Pod更新oldPods中的所有Pod，并将更新后的结果保存到pods参数中
	updatePodsFunc := func(newPods []*v1.Pod, oldPods, pods map[types.UID]*v1.Pod) {
        // 先处理newPods，合并其中相同名字的pod
		filtered := filterInvalidPods(newPods, source, s.recorder)
        // 遍历newPods
		for _, ref := range filtered {
			// Annotate the pod with the source before any comparison.
			if ref.Annotations == nil {
				ref.Annotations = make(map[string]string)
			}
			ref.Annotations[kubetypes.ConfigSourceAnnotationKey] = source
            // 如果newPod已在oldPods中存在
			if existing, found := oldPods[ref.UID]; found {
                // 则将该已存在的pod检查更新之后，保存到pods中
				pods[ref.UID] = existing
				needUpdate, needReconcile, needGracefulDelete := checkAndUpdatePod(existing, ref)
                // 将newPod追加到相应的切片中
				if needUpdate {
					updatePods = append(updatePods, existing)
				} else if needReconcile {
					reconcilePods = append(reconcilePods, existing)
				} else if needGracefulDelete {
					deletePods = append(deletePods, existing)
				}
				continue
			}
			// 在annotation中记录该Pod第一次被应用的时间
			recordFirstSeenTime(ref)
            // 如果newPod在oldPods中不存在，则直接保存到pods中，并追加到addPods中
			pods[ref.UID] = ref
			addPods = append(addPods, ref)
		}
	}

    // 来源channel中传入的其实是PodUpdate对象
	update := change.(kubetypes.PodUpdate)
	switch update.Op {
	case kubetypes.ADD, kubetypes.UPDATE, kubetypes.DELETE:
        // 如果是增删改操作，直接调用updatePodsFunc即可
		if update.Op == kubetypes.ADD {
			klog.V(4).InfoS("Adding new pods from source", "source", source, "pods", klog.KObjs(update.Pods))
		} else if update.Op == kubetypes.DELETE {
			klog.V(4).InfoS("Gracefully deleting pods from source", "source", source, "pods", klog.KObjs(update.Pods))
		} else {
			klog.V(4).InfoS("Updating pods from source", "source", source, "pods", klog.KObjs(update.Pods))
		}
		updatePodsFunc(update.Pods, pods, pods)

	case kubetypes.REMOVE:
		klog.V(4).InfoS("Removing pods from source", "source", source, "pods", klog.KObjs(update.Pods))
        // 如果是REMOVE操作，直接删除pods中相应的Pod
		for _, value := range update.Pods {
			if existing, found := pods[value.UID]; found {
				// this is a delete
				delete(pods, value.UID)
				removePods = append(removePods, existing)
				continue
			}
			// this is a no-op
		}

	// kubelet中三来源均采用SET来推送当前的全量数据
	case kubetypes.SET:
        // 如果是SET操作，需要标记来源
		klog.V(4).InfoS("Setting pods for source", "source", source)
		s.markSourceSet(source)
		// Clear the old map entries by just creating a new map
		oldPods := pods
		pods = make(map[types.UID]*v1.Pod)
		// 把缓存中的数据作为旧数据，推动的全量数据作为新数据，传递到updatePodsFunc中
		// 结果数据传入空map
		updatePodsFunc(update.Pods, oldPods, pods)
        // 将更新后的pods于更新前的进行比较，REMOVE掉在oldPods中但不在pods中的多余的Pod
		for uid, existing := range oldPods {
			if _, found := pods[uid]; !found {
				// this is a delete
				removePods = append(removePods, existing)
			}
		}

	default:
		klog.InfoS("Received invalid update type", "type", update)

	}

	s.pods[source] = pods

	adds = &kubetypes.PodUpdate{Op: kubetypes.ADD, Pods: copyPods(addPods), Source: source}
	updates = &kubetypes.PodUpdate{Op: kubetypes.UPDATE, Pods: copyPods(updatePods), Source: source}
	deletes = &kubetypes.PodUpdate{Op: kubetypes.DELETE, Pods: copyPods(deletePods), Source: source}
	removes = &kubetypes.PodUpdate{Op: kubetypes.REMOVE, Pods: copyPods(removePods), Source: source}
	reconciles = &kubetypes.PodUpdate{Op: kubetypes.RECONCILE, Pods: copyPods(reconcilePods), Source: source}

	return adds, updates, deletes, removes, reconciles
}
```

## checkAndUpdatePod
比较本地缓存和来源中传入的PodUpdate中Pod的区别，返回改动类型，并且将ref(新状态)更新到existing(旧状态)
1. 如果是个有意义的改动，needUpdate=true
2. 如果是个有意义的改动并且是优雅删除，needGracefulDelete=true
3. 如果是个无意义的改动，但是更新了Pod的状态，needReconcile=true
4. 否则全部返回false
```go
// pkg/kubelet/config/config.go
func checkAndUpdatePod(existing, ref *v1.Pod) (needUpdate, needReconcile, needGracefulDelete bool) {

	// 1. this is a reconcile
	// TODO: it would be better to update the whole object and only preserve certain things
	//       like the source annotation or the UID (to ensure safety)
	// 判断是否为有意义的更改
	if !podsDifferSemantically(existing, ref) {
		// this is not an update
		// Only check reconcile when it is not an update, because if the pod is going to
		// be updated, an extra reconcile is unnecessary
		if !reflect.DeepEqual(existing.Status, ref.Status) {
			// Pod with changed pod status needs reconcile, because kubelet should
			// be the source of truth of pod status.
			// 不是有意义的改动并且状态相同，则视为状态更新
			existing.Status = ref.Status
			needReconcile = true
		}
		return
	}

	// Overwrite the first-seen time with the existing one. This is our own
	// internal annotation, there is no need to update.
	ref.Annotations[kubetypes.ConfigFirstSeenAnnotationKey] = existing.Annotations[kubetypes.ConfigFirstSeenAnnotationKey]

	// 先更新
	existing.Spec = ref.Spec
	existing.Labels = ref.Labels
	existing.DeletionTimestamp = ref.DeletionTimestamp
	existing.DeletionGracePeriodSeconds = ref.DeletionGracePeriodSeconds
	existing.Status = ref.Status
	updateAnnotations(existing, ref)

	// 判断是否为优雅删除
	// 2. this is an graceful delete
	if ref.DeletionTimestamp != nil {
		needGracefulDelete = true
	} else {
		// 3. this is an update
		needUpdate = true
	}

	return
}

// 如果Pod的Spec，Labels，DeletionTimestamp，DeletionGracePeriodSeconds，Annotations发生变化。则认为是一次有意义的更改。
func podsDifferSemantically(existing, ref *v1.Pod) bool {
	if reflect.DeepEqual(existing.Spec, ref.Spec) &&
		reflect.DeepEqual(existing.Labels, ref.Labels) &&
		reflect.DeepEqual(existing.DeletionTimestamp, ref.DeletionTimestamp) &&
		reflect.DeepEqual(existing.DeletionGracePeriodSeconds, ref.DeletionGracePeriodSeconds) &&
		isAnnotationMapEqual(existing.Annotations, ref.Annotations) {
		return false
	}
	return true
}
```

## MergedState
MergedState是`pkg/util/config/config.go$Accessor`接口的实现，用于获取当前所有缓存
```go
// pkg/kubelet/config/config.go
func (s *podStorage) MergedState() interface{} {
	s.podLock.RLock()
	defer s.podLock.RUnlock()
	pods := make([]*v1.Pod, 0)
	for _, sourcePods := range s.pods {
		for _, podRef := range sourcePods {
			// 将所有Pod深拷贝
			pods = append(pods, podRef.DeepCopy())
		}
	}
	return pods
}
```

## Sync
重新同步所有缓存到updates
```go
// pkg/kubelet/config/config.go
func (s *podStorage) Sync() {
	s.updateLock.Lock()
	defer s.updateLock.Unlock()
	// 将所有Pod的深拷贝以SET操作发送到updates channel
	s.updates <- kubetypes.PodUpdate{Pods: s.MergedState().([]*v1.Pod), Op: kubetypes.SET, Source: kubetypes.AllSource}
}
```
