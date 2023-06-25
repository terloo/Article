# Secret
Secret解决了密码、token、密钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中。Secret可以以Volume或者环境变量的方式使用。

## Secret的三种类型
1. Service Account：用来访问Kubernets API，由Kubernets自动创建，并且会自动挂载到Pod的`/run/secret/kubernets.io/serviceaccout`目录中。
2. Opaque：base64编码Secret，用来存储密码密钥等。
3. Kubernets.io/dockerconfigjson：用来存储私有docker registry的认证信息。

## Service Account
目录下拥有三个文件
- ca.crt：用于与apiServer传递信息的https证书
- namespace：命名空间
- token：访问apiServer的认证token

## Opaque Secret
1. 创建
  Opaque数据类型为Map，要求Value是base64编码格式
  ```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: myOpaque
  type: Opaque
  data:
    password: xxxxx  # 必须经过base64编码
    username: xxxxx  # 必须经过base64编码
  ```

1. 使用方式
  1. 将Secret挂载到volume
     将会把Secret中所有的数据保存为文件，文件名为key值，文件内容为value值，并且将value解密。
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: myOpaque
       labels:
         app: opaque
     spec:
       containers:
       - name: nginx1
         image: nginx
         volumeMounts:
         - name: secrets
           path: /etc/secret
           readOnly: true
       volumes:
       - name: secrets
         secret:
           secretName: myOpaque
     ```
  
  2. 将Secret导入到环境变量中
     ```yaml
     apiVersion: v1
     kind: Deployment
     metadata:
       name: opaque-deployment-env
     spec:
       replicas: 2
       selector:
         matchLabels:
           app: nginx-secret
       templates:
         metadata: 
           app: nginx-secret
         spec:
           containers:
           - name: nginx
             image: 'nginx:1.7.9'
             ports:
               containerPort: 80
             env:
             - name: TEST-SECRET
               valueFrom:
                 secretKeyRef:
                   name: myOpaque
                   key: username
             envFrom:
             - secretRef:
               name: myOpaque2
     ```

2. Kubernets.io/dockerconfigjson
  1. 使用kubectl创建docker registry认证的secret
     ```shell
     kubectl create secret docker-registry [secret-name] --docker-server=[haber_domain] --docker-username=[username] --docker-password=[password] --docker-email=[email] 
     ```
  2. 创建Pod时引用docker registrt
     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       name: docker-registry
     spec:
       containers:
       - name: nginx
         image: hub.csj.com/test/myapp:v1
       imagePullSecret:
         - name: [secret-name]
     ```
