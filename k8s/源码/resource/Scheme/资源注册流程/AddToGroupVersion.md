# AddToGroupVersion
AddToGroupVersion是metav1包下一个用于注册所有meta资源、版本通用资源、通用转化函数和WatchEvent资源的函数

## 函数
接收一个GV和Scheme，向Scheme中注册资源和转化函数
```go
// vendor/k8s.io/apimachinery/pkg/apis/meta/v1/register.go
func AddToGroupVersion(scheme *runtime.Scheme, groupVersion schema.GroupVersion) {
	scheme.AddKnownTypeWithName(groupVersion.WithKind(WatchEventKind), &WatchEvent{})
	scheme.AddKnownTypeWithName(
		schema.GroupVersion{Group: groupVersion.Group, Version: runtime.APIVersionInternal}.WithKind(WatchEventKind),
		&InternalEvent{},
	)
	// Supports legacy code paths, most callers should use metav1.ParameterCodec for now
	scheme.AddKnownTypes(groupVersion, optionsTypes...)
	// Register Unversioned types under their own special group
	scheme.AddUnversionedTypes(Unversioned,
		&Status{},
		&APIVersions{},
		&APIGroupList{},
		&APIGroup{},
		&APIResourceList{},
	)

	// register manually. This usually goes through the SchemeBuilder, which we cannot use here.
	utilruntime.Must(RegisterConversions(scheme))  // 注册通用转化函数
	utilruntime.Must(RegisterDefaults(scheme))  // 暂无通用默认函数
}

var optionsTypes = []runtime.Object{
	&ListOptions{},
	&GetOptions{},
	&DeleteOptions{},
	&CreateOptions{},
	&UpdateOptions{},
	&PatchOptions{},
}

```

## 注册的资源和函数类型
传入一个GV，向Scheme中注册以下三类资源和转化函数
1. Watch相关资源
   1. G.V.WatchEvent
   2. G.内部版本.InternalEvent
2. Options资源，注意是将其注册为普通资源
   1. G.V.ListOptions
   2. G.V.GetOptions
   3. G.V.DeleteOptions
   4. G.V.CreateOptions
   5. G.V.UpdateOptions
   6. G.V.PatchOptions
3. 无版本资源(幂等)
   1. Status
   2. APIVersions
   3. APIGroupList
   4. APIGroup
   5. APIResourceList
4. 转化函数(幂等)
   1. 基础类型之间的转化
   2. XXXOptions到map[string][]string：用于将XXXOptions转为查询参数
   3. 其他类型之间转化(比如Time)
