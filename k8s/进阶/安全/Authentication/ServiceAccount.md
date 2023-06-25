# ServiceAccount
ServiceAccount是k8s提供的，用于集群中应用进行Authentication的方式，使用的验证方式是Token方式

## 永久Token
ServiceAccount被创建后，会自动创建一个对应的Secret(<=1.24版本)，Secret中保存了一个永久有效的Token，供应用使用
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-sa
```
```sh
$ kubectl get secret
NAME                  TYPE                                  DATA   AGE
test-sa-token-rd9wk   kubernetes.io/service-account-token   3      4d
```

## 临时Token
1.25版本之后，创建ServerAccount将不会自动创建对应的Secret，但仍能手动创建Type为`kubernetes.io/service-account-token`，注解`kubernetes.io/service-account.name: ${ServerAccountName}`的Secret，apiServer会为其附上data数据  
也可以通过kubectl命令直接调用token api申请ServiceAccount对应的Token(具有有效期)
```sh
$ kubectl create token test-sa --duration 10m
eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1TT1lIM3F2TVdtNXNqU3BrVWtHaWl0ZDUwcVVjbVdhTWE4aUFTMFBxcjAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg0NTgwNzA0LCJpYXQiOjE2ODQ1NzcxMDQsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJteWNvbnRyb2xsZXItc3lzdGVtIiwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRlZmF1bHQiLCJ1aWQiOiI2Zjc2MDMwMS1lYjJiLTQyYTItYTZhNi0wOGIzY2ExNTMzNGEifX0sIm5iZiI6MTY4NDU3NzEwNCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Om15Y29udHJvbGxlci1zeXN0ZW06ZGVmYXVsdCJ9.MCeGg8jtcJlj01JY3nlWxiSjbSydjkmDqQ3oDqAVH5XpGxXBcQOImveLV-0MHyYoA59TMPRyopZD_aw9QKKyuDH_ICoC--glNH6LAuIacOOLfDBdBGzBYXh2TseYub1UIxwrWSjvezcZ6RZ-coNjAHa6XEj1zNEAoMQwakuVkb63082leV42C2bP1K-pZqAb7zGl51REgovSsGWpxt4VqZyn9V1my7o9vnNkxl3AJD10-lqNXGKWq4PfMTafp5Lr6DN5C08epslwWW_LQLiNiO2nAw_tSCIEPemANPsuO-decuZDzELWxwN-vKZidJGDJOLT50vXVM2JeC9qvP14Dg
```
kubelet会负责临时Token的申请与刷新，并将其挂载到Pod中

## 挂载(Projected)
serviceAccount通过projected(投射)的容器卷类型将token挂载到容器中  
serviceAccountToken是一种特殊的卷，由kubelet直接生成并挂载，并负责后续刷新
```yaml
# Pod清单
apiVersion: v1
kind: Pod
metadata:
  name: sa-pod
  namespace: default
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: sa-image
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-nvwzc
      readOnly: true
  serviceAccount: test-sa
  serviceAccountName: test-sa
  volumes:
  - name: kube-api-access-nvwzc
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607 # 可以更改这个值来修改过期时间(非刷新时间)
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

## 受众
在生成临时token时，可以指定token的`audience`(受众)(默认为`https://kubernetes.default.svc.cluster.local`)，apiServer以及其他服务(通过TokenReview对象进行校验)可以只接受指定`audience`的token，通过这个机制可以拒绝某些应用的api请求

## ServiceAccount挂载的资源
1. serviceAccountToken：包含kubelet从apiserver获取的令牌，kubelet使用TokenRequest API获取有时间限制的令牌(expirationSeconds)，并将其audience(受众)设置为与api-server的audience匹配。Pod被删除后token立即过期
2. configMap：ca证书
3. downwardAPI：namespace名

## TokenReview
在获取一个Token字符串后，可以创建`TokenReview`对象(非命名空间资源)，在创建时直接输出对象状态进行校验(该对象无法使用get list)，通过`TokenReview`对象，其他服务也可以将apiServer当作集群认证中心来认证请求的身份
```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenReview
metadata:
  name: test
spec:
  token: 填入Token
```
k8s对Token进行校验
```sh
kubectl apply -f tokenReview.yaml -o yaml
```
```yaml
apiVersion: authentication.k8s.io/v1
kind: TokenReview
metadata:
  name: test
spec:
  token: eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1TT1lIM3F2TVdtNXNqU3BrVWtHaWl0ZDUwcVVjbVdhTWE4aUFTMFBxcjAifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg0NTg1MjM5LCJpYXQiOjE2ODQ1ODE2MzksImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJteWNvbnRyb2xsZXItc3lzdGVtIiwic2VydmljZWFjY291bnQiOnsibmFtZSI6Im15Y29udHJvbGxlci1jb250cm9sbGVyLW1hbmFnZXIiLCJ1aWQiOiJkNTQ3MWFkYy1hMWUzLTQzYmYtOTUxNC03MDM1Zjk4NjkzODkifX0sIm5iZiI6MTY4NDU4MTYzOSwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Om15Y29udHJvbGxlci1zeXN0ZW06bXljb250cm9sbGVyLWNvbnRyb2xsZXItbWFuYWdlciJ9.o79xihSZ7lzYenPkFq8prhX04Z2CcOZ8YBEAcb_a4KPrqobF590w-sgJIUDUJbj7lXPQLScJssSfttjIQL5j3QgiiLWe8Bn-uLyc9_CHKfqBOdoZZ5yGEDPOgAtFMNSguT6lGKQtyGMlJ1G4SgJeieRnawdvqKfkI86jSWBTAMk5iNLmDQ5kcGs7mMONg6LsJXg5v6DKuaJn0EruCtkEUmg-cBCa2l1rCDeRNEZS6oLdrkI72nwjiXdPQXCMEAPn6TdK8TVUN1YGNb6CUiXm20kiJYAYTNdXjB3CmTZPTG44x_9ECvLFNpHIyvOXPWYYBJ4snoN-Y-WksIgNKUVolw
status:
  audiences:
  - https://kubernetes.default.svc.cluster.local
  authenticated: true
  user:
    groups:
    - system:serviceaccounts
    - system:serviceaccounts:mycontroller-system
    - system:authenticated
    uid: d5471adc-a1e3-43bf-9514-7035f9869389
    username: system:serviceaccount:mycontroller-system:mycontroller-controller-manager
```