# StorageClass

1. 使用环境：
   1. 通常情况下，使用PVC时需要事先创建好PV，可以使用StorageClass在PVC申请PV时，临时创建并向其分配一个PV。
   2. 以NFS为例，需要一个nfs-client的自动装载程序，称之为Provisioner
      1. 自动创建的PV会以${namespace}-${pvcName}-${pvName}的目录格式存放在NFS服务器中
      2. 如果这个PV被回收，将会将其备份为archieved-${namespace}-${pvcName}-${pvName}的目录格式

2. 创建ServiceAccount为nfs-client进行授权
```yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-clusterrole
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nfs-client-provisioner-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: nfs-client-provisioner
  namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-clusterrole
  apiGroup: rbac.authorization.k8s.io

```

3. 创建nfs-client
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-prosioner
spec:
  replicas: 1
  selector:
    matchLabels:
        app: nfs-client-prosioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-prosioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
      - name: nfs-client-prosioner
        image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
        env:
        - name: PROVISIONER_NAME
          value: nfs-provisioner                     # provisioner名，需要在storageclass中进行引用
        - name: NFS_SERVER
          value: 172.23.238.243                      # NFS服务器地址
        - name: NFS_PATH
          value: /nfs/data/storageclass              # NFS路径
      volumes:
      - name: nfs-client-root
        nfs:
          server: 172.23.238.243
          path: /nfs/data/storageclass
```

4. 创建StorageClass
```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client-storageclass
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" # 是否为默认存储
provisioner: nfs-provisioner                            # provisioner名
parameters:
  archiveOnDelete: "true"  ## 删除pv的时候，pv的内容是否要备份
```

5. 创建PVC引用StorageClass
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-nfs-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-client-storageclass"  # 非必须
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

6. 创建pod使用此PVC
```yml
apiVersion: v1
kind: Pod
metadata:
  name: test-storageclass-pod
spec:
  containers:
  - name: busybox
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    command:
    - "/bin/sh"
    - "-c"
    args:
    - "sleep 3600"
    volumeMounts:
    - name: nfs-pvc
      mountPath: /mnt
  restartPolicy: Never
  volumes:
  - name: nfs-pvc
    persistentVolumeClaim:
      claimName: storage-nfs-pvc
```
