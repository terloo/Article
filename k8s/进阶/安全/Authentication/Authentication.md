# Authentication
验证客户端的身份是否合法

## 用户
k8s支持两种类型的用户
1. 内部用户：由k8s来定义认证信息规范并进行创建
2. 外部用户：非k8s创建的用户
   1. 集群管理员发送的静态Token或者用户证书
   2. 使用Keystone、Google Account以及LDAP等进行认证的用户

## 外部用户认证
用户身份被保存在集群之外，用户使用Token向apiServer发送请求，k8s向外部信息源请求Token有效性并获取用户信息
1. 静态Token文件
2. X.509客户端证书
3. OpenID
4. 认证代理
5. Webhook

## 内部用户认证
k8s内部用户使用`ServiceAccount`的概念来表述，`ServiceAccountl`通过apiServer进行创建并分配给应用。

## 认证方式分类
1. HTTP Token认证  
HTTP Token认证使用一串很长的、难以被模仿的字符串来识别客户的身份。  
每一个Token对应着一个用户，存储在API Server能访问的数据源中(不在etcd中)  
当客户端发起HTTP请求时需要附上请求头`Authorization: Bearer ${TOKEN}`

2. HTTP Base认证  
将`用户名:密码`使用base64编码后放入Authentication请求头中，客户端收到请求后将其解码确定用户身份。

3. HTTPS认证（最严格）  
基于CA根证书签名的**双向**证书客户端认证方式。
   1. 证书颁发
      - 手动签发：通过k8s集群跟ca进行签发HTTPS证书
      - 自动签发：kubelet首次访问API Server时使用Token做认证。通过认证后，Controller Manager会为其颁发证书，以后均采用证书进行HTTPS访问。
   2. kubeconfig
      kubeconfig文件包含了集群的参数（CA证书，API Server地址）、客户端参数（上面生成的证书和私钥）、集群context信息（集群名称，用户名）。kubernetes组件通过启动时指定不同的kubeconfig文件可以切换到不同的集群。
   3. ServiceAccount
      Pod中的容器访问API Server。因为Pod的创建、销毁是动态的，所以不能为其分配证书。kubernetes使用了Service Account解决Pod访问API Server的认证问题。
   4. Secret与SA的关系
      Kubernetes设计了一种资源对象为Secret。Secret分为两类，一种是用于ServiceAccount的service-account-token，另一种是用于保存用户自定义秘密信息的Opaque。ServiceAccount被挂载到所有需要访问API Server的容器的`/run/secrets/serviceaccount/`目录下。ServiceAccount中用到包含三个部分：Token、ca.crt、namespace。
      - token是使用API Server私钥签名的JWT。用于访问API Server时，Server端做认证。
      - ca.crt，根证书。用于Client端验证API Server发送的证书。
      - namespace，标识这个service-account-token的作用域（命名空间）

## 各组件认证方式
1. kubelet、kubectl、ControllerManage、Scheduler、kube-proxy：都通过Https双向认证(kubeconfig文件)
2. Pod：Token(ServiceAccount)