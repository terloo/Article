# PodManager
PodManager缓存着该节点拥有的所有的Pod的缓存，当Pod的缓存产生主动或被动性的变动时，kubelet会更新其中的缓存。
PodManager也维护着`static pod`到`mirror pod`之间的映射关系

## mirror pod
从非apiServer方式获取的Pod被称为`static pod`，`static pod`是无法被apiServer所感知的，为了监控这些pod的状态，kubelet会为每一个`static pod`创建一个对应的`mirror pod`  
`mirror pod`与`static pod`拥有相同的namespace和name，但其他metadata可能不同(如UID)。  
kubelet使用pod全面来报告pod的状态，所以`mirror pod`的状态反映了`static pod`的状态，当`static pod`移除时`mirror pod`也会被移除


## 接口
扩展了MirrorClient接口
```go
// pkg/kubelet/pod/pod_manager.go
type Manager interface {
	// GetPods returns the regular pods bound to the kubelet and their spec.
	GetPods() []*v1.Pod
	// GetPodByFullName returns the (non-mirror) pod that matches full name, as well as
	// whether the pod was found.
	GetPodByFullName(podFullName string) (*v1.Pod, bool)
	// GetPodByName provides the (non-mirror) pod that matches namespace and
	// name, as well as whether the pod was found.
	GetPodByName(namespace, name string) (*v1.Pod, bool)
	// GetPodByUID provides the (non-mirror) pod that matches pod UID, as well as
	// whether the pod is found.
	GetPodByUID(types.UID) (*v1.Pod, bool)
	// GetPodByMirrorPod returns the static pod for the given mirror pod and
	// whether it was known to the pod manager.
	GetPodByMirrorPod(*v1.Pod) (*v1.Pod, bool)
	// GetMirrorPodByPod returns the mirror pod for the given static pod and
	// whether it was known to the pod manager.
	GetMirrorPodByPod(*v1.Pod) (*v1.Pod, bool)
	// GetPodsAndMirrorPods returns the both regular and mirror pods.
	GetPodsAndMirrorPods() ([]*v1.Pod, []*v1.Pod)
	// SetPods replaces the internal pods with the new pods.
	// It is currently only used for testing.
	SetPods(pods []*v1.Pod)
	// AddPod adds the given pod to the manager.
	AddPod(pod *v1.Pod)
	// UpdatePod updates the given pod in the manager.
	UpdatePod(pod *v1.Pod)
	// DeletePod deletes the given pod from the manager.  For mirror pods,
	// this means deleting the mappings related to mirror pods.  For non-
	// mirror pods, this means deleting from indexes for all non-mirror pods.
	DeletePod(pod *v1.Pod)
	// GetOrphanedMirrorPodNames returns names of orphaned mirror pods
	GetOrphanedMirrorPodNames() []string
	// TranslatePodUID returns the actual UID of a pod. If the UID belongs to
	// a mirror pod, returns the UID of its static pod. Otherwise, returns the
	// original UID.
	//
	// All public-facing functions should perform this translation for UIDs
	// because user may provide a mirror pod UID, which is not recognized by
	// internal Kubelet functions.
	TranslatePodUID(uid types.UID) kubetypes.ResolvedPodUID
	// GetUIDTranslations returns the mappings of static pod UIDs to mirror pod
	// UIDs and mirror pod UIDs to static pod UIDs.
	GetUIDTranslations() (podToMirror map[kubetypes.ResolvedPodUID]kubetypes.MirrorPodUID, mirrorToPod map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID)
	// IsMirrorPodOf returns true if mirrorPod is a correct representation of
	// pod; false otherwise.
	IsMirrorPodOf(mirrorPod, pod *v1.Pod) bool

	MirrorClient
}
```

## 实现类
basicManager是PodManager的实现类，并且还负责使用secretManager和configmapManager
```go
// pkg/kubelet/pod/pod_manager.go
// 结构体中的字段都是private的，通过SetPods, AddPod, UpdatePod, or DeletePod 进行修改
type basicManager struct {
	// Protects all internal maps.
	lock sync.RWMutex

	// Regular pods indexed by UID.
	podByUID map[kubetypes.ResolvedPodUID]*v1.Pod
	// Mirror pods indexed by UID.
	mirrorPodByUID map[kubetypes.MirrorPodUID]*v1.Pod

	// Pods indexed by full name for easy access.
	podByFullName       map[string]*v1.Pod
	mirrorPodByFullName map[string]*v1.Pod

	// Mirror pod UID to pod UID map.
	translationByUID map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID

	// basicManager is keeping secretManager and configMapManager up-to-date.
	secretManager    secret.Manager
	configMapManager configmap.Manager

	// A mirror pod client to create/delete mirror pods.
	MirrorClient
}
```
1. secretManager
2. configMapManager
3. podByUID：UID到普通Pod的映射
4. mirrorPodByUID：UID到mirror Pod的映射
5. podByFullName：Pod全名到Pod的映射
6. mirrorPodByFullName：Pod全名到mirror Pod的映射
7. translationByUID：mirror Pod的UID到对应的普通的Pod的映射

## 构造函数
```go
// pkg/kubelet/pod/pod_manager.go
func NewBasicPodManager(client MirrorClient, secretManager secret.Manager, configMapManager configmap.Manager) Manager {
	pm := &basicManager{}
	pm.secretManager = secretManager
	pm.configMapManager = configMapManager
	pm.MirrorClient = client
	// 初始化，将所有Pod设置为nil，仅在初始化和测试时使用
	pm.SetPods(nil)
	return pm
}

func (pm *basicManager) SetPods(newPods []*v1.Pod) {
	pm.lock.Lock()
	defer pm.lock.Unlock()

	pm.podByUID = make(map[kubetypes.ResolvedPodUID]*v1.Pod)
	pm.podByFullName = make(map[string]*v1.Pod)
	pm.mirrorPodByUID = make(map[kubetypes.MirrorPodUID]*v1.Pod)
	pm.mirrorPodByFullName = make(map[string]*v1.Pod)
	pm.translationByUID = make(map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID)

	pm.updatePodsInternal(newPods...)
}
```


## AddPod UpdatePod
对Pod缓存的增改最终都是通过updatePodsInternal进行实现的  
对Pod的处理需要对Pod的中Secret和Configmap进行同步处理
```go
// pkg/kubelet/pod/pod_manager.go
func (pm *basicManager) AddPod(pod *v1.Pod) {
	pm.UpdatePod(pod)
}

func (pm *basicManager) UpdatePod(pod *v1.Pod) {
	pm.lock.Lock()
	defer pm.lock.Unlock()
	pm.updatePodsInternal(pod)
}

// 更新pod状态
func (pm *basicManager) updatePodsInternal(pods ...*v1.Pod) {
	for _, pod := range pods {
		// pod如果已经结束，则取消Pod在secretManager和configMapManager中的注册
		if pm.secretManager != nil {
			if isPodInTerminatedState(pod) {
				// Pods that are in terminated state and no longer running can be
				// ignored as they no longer require access to secrets.
				// It is especially important in watch-based manager, to avoid
				// unnecessary watches for terminated pods waiting for GC.
				pm.secretManager.UnregisterPod(pod)
			} else {
				// TODO: Consider detecting only status update and in such case do
				// not register pod, as it doesn't really matter.
				pm.secretManager.RegisterPod(pod)
			}
		}
		if pm.configMapManager != nil {
			if isPodInTerminatedState(pod) {
				// Pods that are in terminated state and no longer running can be
				// ignored as they no longer require access to configmaps.
				// It is especially important in watch-based manager, to avoid
				// unnecessary watches for terminated pods waiting for GC.
				pm.configMapManager.UnregisterPod(pod)
			} else {
				// TODO: Consider detecting only status update and in such case do
				// not register pod, as it doesn't really matter.
				pm.configMapManager.RegisterPod(pod)
			}
		}
		// 更新podManager中所有的map
		podFullName := kubecontainer.GetPodFullName(pod)
		// This logic relies on a static pod and its mirror to have the same name.
		// It is safe to type convert here due to the IsMirrorPod guard.
		if kubetypes.IsMirrorPod(pod) {
			mirrorPodUID := kubetypes.MirrorPodUID(pod.UID)
			pm.mirrorPodByUID[mirrorPodUID] = pod
			pm.mirrorPodByFullName[podFullName] = pod
			if p, ok := pm.podByFullName[podFullName]; ok {
				pm.translationByUID[mirrorPodUID] = kubetypes.ResolvedPodUID(p.UID)
			}
		} else {
			resolvedPodUID := kubetypes.ResolvedPodUID(pod.UID)
			// 如果是普通的Pod，还需要传入新旧Pod来更新metrics
			updateMetrics(pm.podByUID[resolvedPodUID], pod)
			pm.podByUID[resolvedPodUID] = pod
			pm.podByFullName[podFullName] = pod
			if mirror, ok := pm.mirrorPodByFullName[podFullName]; ok {
				pm.translationByUID[kubetypes.MirrorPodUID(mirror.UID)] = resolvedPodUID
			}
		}
	}
}

func isPodInTerminatedState(pod *v1.Pod) bool {
	return pod.Status.Phase == v1.PodFailed || pod.Status.Phase == v1.PodSucceeded
}
```

## DeletePod
```go
// pkg/kubelet/pod/pod_manager.go
func (pm *basicManager) DeletePod(pod *v1.Pod) {
	// 新Pod为nil
	updateMetrics(pod, nil)
	pm.lock.Lock()
	defer pm.lock.Unlock()
	// 取消注册
	if pm.secretManager != nil {
		pm.secretManager.UnregisterPod(pod)
	}
	if pm.configMapManager != nil {
		pm.configMapManager.UnregisterPod(pod)
	}
	podFullName := kubecontainer.GetPodFullName(pod)
	// It is safe to type convert here due to the IsMirrorPod guard.
	if kubetypes.IsMirrorPod(pod) {
		mirrorPodUID := kubetypes.MirrorPodUID(pod.UID)
		delete(pm.mirrorPodByUID, mirrorPodUID)
		delete(pm.mirrorPodByFullName, podFullName)
		delete(pm.translationByUID, mirrorPodUID)
	} else {
		delete(pm.podByUID, kubetypes.ResolvedPodUID(pod.UID))
		delete(pm.podByFullName, podFullName)
	}
}
```

## 对mirrorPod的一些判断方法
```go
// pkg/kubelet/pod/pod_manager.go

// 将普通Pod的UID转化为mirrorPod的UID
func (pm *basicManager) TranslatePodUID(uid types.UID) kubetypes.ResolvedPodUID {
	// It is safe to type convert to a resolved UID because type conversion is idempotent.
	if uid == "" {
		return kubetypes.ResolvedPodUID(uid)
	}

	pm.lock.RLock()
	defer pm.lock.RUnlock()
	if translated, ok := pm.translationByUID[kubetypes.MirrorPodUID(uid)]; ok {
		return translated
	}
	return kubetypes.ResolvedPodUID(uid)
}

// 获取所有普通Pod和mirrorPod UID的映射和反向映射
func (pm *basicManager) GetUIDTranslations() (podToMirror map[kubetypes.ResolvedPodUID]kubetypes.MirrorPodUID,
	mirrorToPod map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()

	podToMirror = make(map[kubetypes.ResolvedPodUID]kubetypes.MirrorPodUID, len(pm.translationByUID))
	mirrorToPod = make(map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID, len(pm.translationByUID))
	// Insert empty translation mapping for all static pods.
	for uid, pod := range pm.podByUID {
		if !kubetypes.IsStaticPod(pod) {
			continue
		}
		podToMirror[uid] = ""
	}
	// Fill in translations. Notice that if there is no mirror pod for a
	// static pod, its uid will be translated into empty string "". This
	// is WAI, from the caller side we can know that the static pod doesn't
	// have a corresponding mirror pod instead of using static pod uid directly.
	for k, v := range pm.translationByUID {
		mirrorToPod[k] = v
		podToMirror[v] = k
	}
	return podToMirror, mirrorToPod
}

// 获取所有mirrorPod的全名
func (pm *basicManager) GetOrphanedMirrorPodNames() []string {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	var podFullNames []string
	for podFullName := range pm.mirrorPodByFullName {
		if _, ok := pm.podByFullName[podFullName]; !ok {
			podFullNames = append(podFullNames, podFullName)
		}
	}
	return podFullNames
}

// 判断一个mirrorPod和普通pod是否对应
func (pm *basicManager) IsMirrorPodOf(mirrorPod, pod *v1.Pod) bool {
	// Check name and namespace first.
	if pod.Name != mirrorPod.Name || pod.Namespace != mirrorPod.Namespace {
		return false
	}
	hash, ok := getHashFromMirrorPod(mirrorPod)
	if !ok {
		return false
	}
	return hash == getPodHash(pod)
}
```

## 其余Get方法
```go
// pkg/kubelet/pod/pod_manager.go

// 获取所有Pod和mirror Pods
func (pm *basicManager) GetPods() []*v1.Pod {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	return podsMapToPods(pm.podByUID)
}

func (pm *basicManager) GetPodsAndMirrorPods() ([]*v1.Pod, []*v1.Pod) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	pods := podsMapToPods(pm.podByUID)
	mirrorPods := mirrorPodsMapToMirrorPods(pm.mirrorPodByUID)
	return pods, mirrorPods
}

func podsMapToPods(UIDMap map[kubetypes.ResolvedPodUID]*v1.Pod) []*v1.Pod {
	pods := make([]*v1.Pod, 0, len(UIDMap))
	for _, pod := range UIDMap {
		pods = append(pods, pod)
	}
	return pods
}

func mirrorPodsMapToMirrorPods(UIDMap map[kubetypes.MirrorPodUID]*v1.Pod) []*v1.Pod {
	pods := make([]*v1.Pod, 0, len(UIDMap))
	for _, pod := range UIDMap {
		pods = append(pods, pod)
	}
	return pods
}

// 从Pod的UID或者全名获取Pod
func (pm *basicManager) GetPodByUID(uid types.UID) (*v1.Pod, bool) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	pod, ok := pm.podByUID[kubetypes.ResolvedPodUID(uid)] // Safe conversion, map only holds non-mirrors.
	return pod, ok
}

func (pm *basicManager) GetPodByName(namespace, name string) (*v1.Pod, bool) {
	podFullName := kubecontainer.BuildPodFullName(name, namespace)
	return pm.GetPodByFullName(podFullName)
}

func (pm *basicManager) GetPodByFullName(podFullName string) (*v1.Pod, bool) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	pod, ok := pm.podByFullName[podFullName]
	return pod, ok
}

// Mirror Pod 到 Pod的转化
func (pm *basicManager) GetMirrorPodByPod(pod *v1.Pod) (*v1.Pod, bool) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	mirrorPod, ok := pm.mirrorPodByFullName[kubecontainer.GetPodFullName(pod)]
	return mirrorPod, ok
}

func (pm *basicManager) GetPodByMirrorPod(mirrorPod *v1.Pod) (*v1.Pod, bool) {
	pm.lock.RLock()
	defer pm.lock.RUnlock()
	pod, ok := pm.podByFullName[kubecontainer.GetPodFullName(mirrorPod)]
	return pod, ok
}
```
