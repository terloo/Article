# kubectl配置文件
kubectl是k8s提供的与api-server进行交互的客户端，需要一个配置文件来指定登陆使用的用户和登陆的集群

## 读取配置文件的位置
1. --kubeconfig：命令行参数指定配置文件位置，不会合并其他配置文件
2. $KUBECONFIG：同时使用此环境变量指定的所有文件列表(操作系统默认文件顺序)，所有文件将被合并。当修改一个值时，将修改设置了该值的文件。当创建一个值时，将在列表的首个文件创建该值。若列表中所有的文件都不存在，将创建列表中的最后一个文件。
3. 如果前两项都没有设置,将使用 ${HOME}/.kube/config，并且不会合并其他文件。

## 配置文件内容
1. cluster：集群
2. user：用户
3. context：用户与集群的绑定关系，多对多

## 配置文件
```yml
apiVersion: v1
kind: Config
current-context: kubernetes-admin@kubernetes
clusters:
- cluster:
    certificate-authority-data: XXXXXXXXX
    server: https://master:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
users:
- name: kubernetes-admin
  user:
    client-certificate-data: XXXXXXXXX
    client-key-data: XXXXXXXXX
preferences: {}
```

## 配置文件说明
1. current-context：当前使用的context
2. clusters[]：当前配置的所有集群信息
3. clusters[].cluster.certificate-authority-data：集群的ca证书
4. users[].user.client-certificate-data：用户的ca证书
5. users[].user.client-key-data：用户的私钥，仅仅保存于此处