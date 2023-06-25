# MirrorClient
MirrorClient是kubelet用于创建mirror pod的客户端

## 接口
```go
// pkg/kubelet/pod/mirror_client.go
type MirrorClient interface {
	// CreateMirrorPod creates a mirror pod in the API server for the given
	// pod or returns an error.  The mirror pod will have the same annotations
	// as the given pod as well as an extra annotation containing the hash of
	// the static pod.
	CreateMirrorPod(pod *v1.Pod) error
	// DeleteMirrorPod deletes the mirror pod with the given full name from
	// the API server or returns an error.
	DeleteMirrorPod(podFullName string, uid *types.UID) (bool, error)
}
```
1. CreateMirrorPod：创建一个mirror pod，mirror pod拥有一个额外的annotation保存着static pod的hash
2. DeleteMirrorPod：根据传入的pod全名，从apiServer中删除该mirror pod

## 实现类
```go
// pkg/kubelet/pod/mirror_client.go
type basicMirrorClient struct {
	apiserverClient clientset.Interface
	nodeGetter      nodeGetter
	nodeName        string
}
```
1. apiserverClient：k8s的clientSet
2. nodeGetter：能获取node或nodeList的getter
3. nodeName：node名

## 构造函数
```go
// pkg/kubelet/pod/mirror_client.go
func NewBasicMirrorClient(apiserverClient clientset.Interface, nodeName string, nodeGetter nodeGetter) MirrorClient {
	return &basicMirrorClient{
		apiserverClient: apiserverClient,
		nodeName:        nodeName,
		nodeGetter:      nodeGetter,
	}
}
```

## CreateMirrorPod
在生成MirrorPod时，会添加一个annotation kubernetes.io/config.mirror的注解，该注解表明了该Pod是否为MirrorPod
```go
// pkg/kubelet/pod/mirror_client.go
func (mc *basicMirrorClient) CreateMirrorPod(pod *v1.Pod) error {
	if mc.apiserverClient == nil {
		return nil
	}
	// Make a copy of the pod.
	copyPod := *pod
	copyPod.Annotations = make(map[string]string)

	for k, v := range pod.Annotations {
		copyPod.Annotations[k] = v
	}
	hash := getPodHash(pod)
    // 添加一个annotation kubernetes.io/config.mirror=pod的hash
	copyPod.Annotations[kubetypes.ConfigMirrorAnnotationKey] = hash

	// With the MirrorPodNodeRestriction feature, mirror pods are required to have an owner reference
	// to the owning node.
	// See https://git.k8s.io/enhancements/keps/sig-auth/1314-node-restriction-pods/README.md
	nodeUID, err := mc.getNodeUID()
	if err != nil {
		return fmt.Errorf("failed to get node UID: %v", err)
	}
	controller := true
    // 添加一个OwnerReferences，kind为Node
	copyPod.OwnerReferences = []metav1.OwnerReference{{
		APIVersion: v1.SchemeGroupVersion.String(),
		Kind:       "Node",
		Name:       mc.nodeName,
		UID:        nodeUID,
		Controller: &controller,
	}}

    // 调用clientSet进行创建
	apiPod, err := mc.apiserverClient.CoreV1().Pods(copyPod.Namespace).Create(context.TODO(), &copyPod, metav1.CreateOptions{})
	if err != nil && apierrors.IsAlreadyExists(err) {
		// Check if the existing pod is the same as the pod we want to create.
		if h, ok := apiPod.Annotations[kubetypes.ConfigMirrorAnnotationKey]; ok && h == hash {
			return nil
		}
	}
	return err
}
```

## DeleteMirrorPod
接收一个pod全名和一个可选的参数uid。当uid不为nil时，仅仅当uid能与传入的pod全名相匹配时才删除该mirror pod，不匹配时返回false而不是一个error
```go
// pkg/kubelet/pod/mirror_client.go
func (mc *basicMirrorClient) DeleteMirrorPod(podFullName string, uid *types.UID) (bool, error) {
	if mc.apiserverClient == nil {
		return false, nil
	}
	name, namespace, err := kubecontainer.ParsePodFullName(podFullName)
	if err != nil {
		klog.ErrorS(err, "Failed to parse a pod full name", "podFullName", podFullName)
		return false, err
	}

	var uidValue types.UID
	if uid != nil {
		uidValue = *uid
	}
	klog.V(2).InfoS("Deleting a mirror pod", "pod", klog.KRef(namespace, name), "podUID", uidValue)

	var GracePeriodSeconds int64
    // 通过uid来删除Pod，GracePeriod为0
	if err := mc.apiserverClient.CoreV1().Pods(namespace).Delete(context.TODO(), name, metav1.DeleteOptions{GracePeriodSeconds: &GracePeriodSeconds, Preconditions: &metav1.Preconditions{UID: uid}}); err != nil {
		// Unfortunately, there's no generic error for failing a precondition
		if !(apierrors.IsNotFound(err) || apierrors.IsConflict(err)) {
			// We should return the error here, but historically this routine does
			// not return an error unless it can't parse the pod name
			klog.ErrorS(err, "Failed deleting a mirror pod", "pod", klog.KRef(namespace, name))
		}
		return false, nil
	}
	return true, nil
}
```

## 其他方法
```go
// pkg/kubelet/pod/mirror_client.go
// 获取NodeUID
func (mc *basicMirrorClient) getNodeUID() (types.UID, error) {
	node, err := mc.nodeGetter.Get(mc.nodeName)
	if err != nil {
		return "", err
	}
	if node.UID == "" {
		return "", fmt.Errorf("UID unset for node %s", mc.nodeName)
	}
	return node.UID, nil
}

// 获取kubernetes.io/config.mirror的值为mirrorPod的hash
func getHashFromMirrorPod(pod *v1.Pod) (string, bool) {
	hash, ok := pod.Annotations[kubetypes.ConfigMirrorAnnotationKey]
	return hash, ok
}

// 获取kubernetes.io/config.hash的值为pod的hash
func getPodHash(pod *v1.Pod) string {
	// The annotation exists for all static pods.
	return pod.Annotations[kubetypes.ConfigHashAnnotationKey]
}
```