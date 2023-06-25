# AdmissionController 准入控制
准入控制是API Server的插件集合，通过添加不同的插件，实现额外的准入控制。甚至API Server的一些主要功能都需要通过Admission Controller实现，比如ServiceAccount

## 作用
1. 安全的角度：
   1. 需要明确在kubernetes集群中部署的镜像来源是否可信，以免遭受攻击
   2. 一般情况下，在Pod中尽量不使用root用户，或者尽量不开启特权容器等
2. 治理的角度：
   1. 比如业务需要通过label对业务/服务进行区分，那么可以通过admission controller校验服务是否已经存在对应的label等
   2. 添加资源配额限制，以免出现资源超卖的情况

## 流程
AdmissionController的过程分为两步，每步都有多个Plugin执行
1. 变更准入控制器(Mutating Admission)：可以修改被它接受的对象，由此引出第二个作用，将相关资源作为请求处理的一部分进行变更
2. 验证准入控制器(Validating Admission)：只能进行验证，不能进行任何资源数据的修改操作

## 注意点
1. 某些控制器可以即是变更准入控制器又是验证准入控制器。
2. 如果任意一个准入控制器拒绝了该请求，则整个请求将会被立即拒绝，并向终端用户返回错误。
3. 请求处理时，要注意处理资源对象的所有API版本

## 扩展
AdmissionController主要通过以下两个Plugin进行扩展
- MutatingWebhook：提供了一种扩展变更准入控制器的方法，可以通过该AdmissionController来访问用户自建服务
- ValidatingWebhook：提供了一种扩展验证准入控制器的方法，可以通过该AdmissionController来访问用户自建服务
> MutatingWebhook和ValidatingWebhook统称为动态准入控制器
> MutatingWebhook的处理是串行的，但是不保证顺序
> ValidatingWebhook的处理是并行的

## 举例
列举几个插件的功能：
- NamespaceLifeCycle：防止在不存在的namespace上创建对象，防止删除系统预置的namespace，删除namespace时，连带删除它所有的资源对象。
- LimitRanger：确保请求的资源不会超过资源所在namespace的LimitRanger限制
- ServiceAccount：实现了自动化添加ServiceAccount
- ResourceQuota：确保请求的资源不会超过资源的ResourceQuota限制
