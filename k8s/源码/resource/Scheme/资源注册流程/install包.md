# install包
在每个资源内部包中，都有一个install/install.go负责调用AddToScheme将该资源版本下所有资源注册到资源注册表中

```go
// pkg/apis/core/install/install.go
func init() {
	Install(legacyscheme.Scheme)
}

func Install(scheme *runtime.Scheme) {
	utilruntime.Must(core.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion))
}
```
1. legacyscheme.Scheme是kube-apiserver组件的全局资源注册表，k8s中所有资源信息都交由资源注册表统一管理。
2. core.AddToScheme注册所有core组内部版本
3. v1.AddToScheme注册所有core组外部版本
4. scheme.SetVersionPriority设置资源组的版本顺序，第一个资源为首选版本

## 版本优先级
k8s中使用`vendor/k8s.io/apimachinery/pkg/runtime/scheme.go#Scheme`结构体来管理所有资源，Scheme中使用versionPriority来存储资源版本优先级
```go
type Scheme struct {
    ...

    // 例如 "apps"{"v1", "v1beta2", "v1beta1"}
    versionPriority map[string][]string

    ...
}
```
1. 使用scheme.PrioritizedVersionsForGroup(groupName string)来获取指定group的所有版本，按优先级排列
2. 使用scheme.PrioritizedVersionsAllGroups()来获取所有group的所有版本，按优先级排列
3. 使用scheme.SetVersionPriority(versions ..schema.GroupVersion)设置资源组的版本顺序，第一个资源为首选版本