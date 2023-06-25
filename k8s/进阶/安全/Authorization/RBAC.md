# RBAC
RBAC是一种基于角色的访问控制框架，相对于其他鉴权框架，拥有以下优势：
- 对集群中的资源和非资源均拥有完整的覆盖
- 整个RBAC完全由几个API对象完成，同其他k8s对象一样，可以使用kubectl和API进行操作
- 可以在运行中调整，无需重启API Server
RBAC引入了4个新的顶级资源对象：Role、ClusterRole、RoleBinding、ClusterRoleBinding，4种对象资源均可通过kubectl与API进行操作
- Role：对资源操作的抽象（比如创建Pod、删除一个Service等），Role是一个名称空间级别的操作。
- ClusterRole：对资源操作的抽象，ClusterRole是集群级别的操作。
- RoleBinding：将Role或者ClusterRole和一个或多个用户、组、SA进行绑定
- ClusterRoleBinding：将Role或者ClusterRole和一个或多个用户、组、SA进行绑定
> 如果用户通过RoleBinding绑定到了一个ClusterRole上，那么用户也只能进行名称空间级别的操作

## 绑定关系
1. 用户(组，sa) -> Role 使用RoleBinding
2. 用户(组，sa) -> ClusterRole 使用ClusterRoleBinding
3. 用户(组，sa) -> ClusterRole 使用RoleBinding：会使得ClusterRole降为命名空间级别的Role
4. 用户(组，sa) -> Role 使用ClusterRoleBinding：无法绑定

## Role
Role表示一组规则权限，权限设置只能采用白名单形式；Role可以定义在一个namespace中，如果要跨namespace则可以创建ClusterRole。
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: authorization-role
rules: 
- apiGroups: [""]  # 资源所属的api组，空字符串代表核心API组
    resources: ["pods"]  # 以复数的形式展现，可以用["pods/log"]这种形式来表示子资源
    verbs: ["get", "watch", "list"]
```
> verbs列表：get, list, watch, create, update, patch, delete, exec

## ClusterRole
- 集群级别的资源控制（如node访问权限）
- 非资源型endpoints（如/healthz访问）
- 所有命名空间级别的资源（如Pods）
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: authorization-clusterrole
rules:
- apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "watch", "list"]
```

## RoleBinding
RoleBinding可以将Role中定义的权限赋予用户或组，RoleBinding包含一组权限列表（subjects），权限列表中包含有不同形式的待授予权限资源类型（users、groups、sa）。  
RoleBinding同样包含对Binding的Role引用；RoleBinding适用于一个命名空间内的授权，而ClusterRoleBinding适用于集群范围内的授权。  
```yaml
# 将上面创建的`authorization-role`授予jane用户，此后jane用户就能在default命名空间下具有`authorization-role`中指定的权限。
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-binding
  namespace: default
subjects:
  - kind: User
    name: jane
    apiGruop: rbac.authorization.k8s.io  # 可以省略，User和Group时默认为rbac组，ac时默认为核心组
roleRef:
  kind: Role
  name: authorization-role
  apiGroup: rbac.authorization.k8s.io
```
```yaml
# 将ClusterRole用RoleBinding绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: role-binding-cluster-role
  namespace: default
subjects:
- kind: User
  name: mary
roleRef:
  kind: ClusterRole
  name: authorization-role
  apiGroup: rbac.authorization.k8s.io
```

## ClusterRoleBinding
ClusterRoleBinding将ClusterRole中定义的权限赋予用户或者组。ClusterRoleBinding中不需要指定namespace。
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-role-binding
subjects:
- kind: User
  name: jane
roleRef:
  kind: ClusterRole
  name: authorization-clusterrole
  apiGroup: rbac.authorization.k8s.io
```