# Auditing
审计是k8s的api-server提供的时序性集群操作记录，每个请求在每个阶段都会生成一个审计事件，并在预处理之后将其写入后端。  

## 审计阶段
审计阶段决定了生成日志的时间点，默认在以下所有时间点生成日志
1. RequestReceived：收到请求后，进入委托handler chain之前
2. ResponseStarted：响应头发送后，响应体发送前，该阶段只适用于长连接
3. 490c71b340fa956a297b3996294ee421
4. Panic：如果事件生成器出现了Panic

## 策略
审计策略定义了什么事件和事件的哪些数据应当被记录
1. None：不记录该事件
2. Metadata：记录事件元数据(用户，请求事件，资源，操作等)，但是不记录请求体和响应体
3. Request：记录事件元数据和请求体，但是不记录响应体。不适用非资源型请求
4. ReqeustResponse：记录事件元数据，请求体，响应体。不适用非资源型请求

## 策略配置文件
当事件发生时，将会按顺序匹配所有的规则，api-server将会根据第一个匹配到的规则来记录时间。使用`--audit-policy-file`选项可以指定配置策略配置文件，如果该选项被省略，则没有事件会被记录。
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# 忽略的审计阶段
omitStages:
- RequestReceived
rules:
- level: RequestResponse
  # 匹配的资源
  resources:
  - group: ""
    resources: ["pods"] # 与rbac类似

# 记录pod的log status的Metadata
- level: Metadata
  resources:
  - group: ""
    resources: ["pods/log", "pods/status"]

# 不记录名为controller-leader的configmap的资源访问
- level: None
  resources:
  - group: ""
    resources: ["configmaps"]
    resourceName: ["controller-leader"]

# 不记录已认证用户对某些非资源URL路径的访问
- level: None
  userGroups: ["system:authenticated"]
  nonResourceURLs:
  - "/api*"   # 通配符匹配
  - "/version"

# 不记录用户system:kube-proxy对endpoints和services资源的watch操作
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: "" # core API group
    resources: ["endpoints", "services"]

# 记录kube-system命名空间下configmaps的修改
- level: Request
  resources:
  - group: ""
    resources: ["configmaps"]
  # 空字符串可用于非命名空间资源
  namespaces: ["kube-system"]

# Log configmap and secret changes in all other namespaces at the Metadata level.
- level: Metadata
  resources:
  - group: "" # core API group
    resources: ["secrets", "configmaps"]

# Log all other resources in core and extensions at the Request level.
- level: Request
  resources:
  - group: "" # core API group
  - group: "extensions" # Version of group should NOT be included.

# 使用元数据级别记录其他所有请求
- level: Metadata
  # Long-running requests like watches that fall under this rule will not
  # generate an audit event in RequestReceived.
  omitStages:
    - "RequestReceived"
```

## 最小化策略配置文件
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 使用元数据级别记录所有的事件
- level: Metadata
```

## 审计存储后端
审计存储后端持久化审计日志到一个外部存储，k8s提供了两种开箱即用的存储后端
1. 本地文件系统
2. Webhook，发送日志到一个外部HTTP API
日志的存储结构体格式为`audit.k8s.io/v1,Event`资源对象，格式为JSON

### 本地文件系统命令行选项
1. --audit-log-path：指定日志文件路径，`-`代表标准输出流。未指定代表关闭本地文件系统存储后端
2. --audit-log-maxage：指定日志文件最长保存时间，单位天
3. --audit-log-maxbackup：指定日志文件最大保存数量
4. --audit-log-maxsize：指定日志文件最大的轮换大小

## 事件处理模式
使用`--audit-webhook-mode`选项可以指定事件的缓冲策略
1. batch：缓存事件到一定数量，然后批量异步处理
2. blocking：每个事件都阻塞当前api来响应处理事件
3. blocking-strict：与blocking类似，当RequestReceived阶段失败时，整个请求都会失败


nohup kube-apiserver --audit-policy-file /root/manifest/audit.yaml --audit-log-path /root/log/audit.log --secure-port=6443 --bind-address 0.0.0.0 --tls-cert-file=/etc/kubernetes/pki/k8s_server.crt --tls-private-key-file=/etc/kubernetes/pki/k8s_server.key --client-ca-file=/etc/kubernetes/pki/ca.crt --apiserver-count=1 --endpoint-reconciler-type=master-count --etcd-servers=https://192.168.100.100:4001 --etcd-cafile=/etc/kubernetes/pki/ca.crt --etcd-certfile=/etc/kubernetes/pki/etcd_client.crt --etcd-keyfile=/etc/kubernetes/pki/etcd_client.key --service-cluster-ip-range=10.88.0.0/16 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --allow-privileged=true --logtostderr=true &> /root/log/kube-apiserver.log &