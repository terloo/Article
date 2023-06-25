# go-restful
k8s使用go-result来构建api-server的rest服务

## go-restful三个核心组件
1. Container：服务器，包含了多个WebService和一个http.ServeMux，使用RouteSelector进行分发
2. WebService：服务，包含了多个Route，并拥有一个根路由路径
3. Route：路由，包含URL、HTTPmethod、输入输出类型、回调函数

## 路由
1. 先根据路由前缀匹配到合适的WebService的跟路由
2. restful.Container通过ServeMux来进行一层路由，该路由在默认情况下不路由，而是直接转发到RouteSelector。
3. 通过restful.RouteSelector(实现类为restful.CurlyRouter)来进行二层路由，RouteSelector主要实现了restful风格的uri路由，将请求路由到匹配的Route
4. k8s的核心api(KubeAPIService)的restful风格都是通过restful包的RouteSelector来进行Handler路由，即ServeMux直接路由到RouteSelector。

## 在k8s中的应用
1. Container
   1. k8s中一共使用了三个Container，分别对应了三个服务
2. WebService
   1. 一个WebSerice对应着一个GV，其根路由为`/apis/<group>/<version>`或者`/api/v1/`(对于核心api)或者
3. Route的路径为，每个资源的每个verb都拥有一个Route
   1. 命名空间资源`/namespace/{namespaceName>}/<resource>/{resourcesName}`
   2. 非命名空间资源`/<resource>/{resourceName}`
