# RESTOptionsGetter
RESTOptionsGetter是k8s中用于获取RESTOptions的接口，Regsitry层根据此结构体来创建自身和Storage层

## 接口
```go
// vendor/k8s.io/apiserver/pkg/registry/generic/options.go
type RESTOptionsGetter interface {
	GetRESTOptions(resource schema.GroupResource) (RESTOptions, error)
}
```

## 实现类
1. RESTOptions：RESTOptions本身也是该接口的实现类，实现方法直接返回自身
2. StorageFactoryRestOptionsFactory：返回**构建KubeAPIServer所管理的k8s内置资源的Service层时**使用的RESTOptions，由于使用了工厂模式来构造RESTOptions，所以可以为不同的资源生成不同的Config
3. CRDRESTOptionsGetter：返回APIExtensionsServer在**构建CRD的Service层时**使用的RESTOptions
4. SimpleRestOptionsFactory：返回**构建非KubeAPIServer所管理的k8s内置资源的Service层时**使用的RESTOptions，即`apiextensions.k8s.io`和`apiregistration.k8s.io`组的资源

## 区别
CRDRESTOptionsGetter和SimpleRestOptionsFactory生成的RESTOptions中的资源存储路径加上了组名  
而StorageFactoryRestOptionsFactory生成的RESTOptions中的资源存储路径通过Factory获取(一般是不加组名)