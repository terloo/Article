# Authorization
认证过程只是确认了通信的双方是可信的，可以相互通信。而鉴权是确定请求方有哪些资源的权限。API Server有以下几种授权策略。（通过API Server的--authorization-mode 启动参数可以设置）
- AlwaysDeny：表示拒绝所有请求，一般用于测试
- AlwaysAllow：允许接收所有的请求，如果集群不需要授权流程，则可以采用该策略。
- ABAC（Attribute-Based Access Control）：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制
- Webhook：通过调用外部ERST服务对用户进行授权
- RBAC（Role-Based Access Control）：基于角色的访问控制，现行默认规则
