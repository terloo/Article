# PV-PVC

## PV
持久化容器卷，提前声明一块空间，可以规定大小，用于pod挂载
1. 配置
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    storage: 30M   # 容量
  accessModes:
  - ReadWriteMany  # 访问模式
  storageClassName: pv-group1   # pv所属的种类，同一种pv可以被pvc挑选
  nfs:
    path: /data/nfs/pv-01
    server: master
```
1. accessModes：PV访问策略控制，可以设置多个访问策略，PVC在绑定PV，只要有一个策略符合即可绑定
   1. ReadWriteOnce：只允许单node读写访问
   2. ReadOnlyMany：允许多node只读访问
   3. ReadWriteMany：允许多node读写访问
2. storageClassName：pv所属的组名
3. persistentVolumeReclaimPolicy：PV被release之后(与其绑定的PVC被删除)再利用策略
   1. Recycle：已废弃
   2. Delete：volume被release之后直接删除，需要volume plugin支持
   3. Retain：保留，由管理员手动处理
4. nodeAffinity：限制可以访问该PV的node，使用该PV的Pod只能调度到满足亲和性的node上

## PV支持的类型
1. hostPath：仅用于单机测试，不能用于生产环境
2. local：类似于hostPath，但必须给该pv配置节点亲和。k8s通过节点亲和性来决定在什么节点上创建卷
3. nfs：nfs网络存储
4. cephfs：ceph网络存储
5. 其他云厂商或者共享存储

## PVC
持久化容器卷声明，pv要绑定至pvc进行使用，绑定之后为一对一关系

1. 创建一个pvc，并声明需要的pv的属性
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  storageClassName: pv-group1  # 指定挑选pv的组
  accessModes:
  - ReadWriteMany              # pv的访问模式
  resources:
    requests:
      storage: 20M             # pv需要的最小空间，将会自动匹配合适的pv
```
2. 在pod中使用pvc
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-pv-demo
  labels:
    app: nginx-pv-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ngxinx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
      name: nginx-pod
    spec:
      containers:
      - image: nginx:1.21.3
        name: nginx-container
        volumeMount:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        persistentVolumeClaim:
          name: pvc-demo
```
3. PVC如果storeClassName被设置为""(`storeClassName=""`)，则表示该PVC不要求特定的Class，系统只选择未设定Class的PV与之匹配
4. PVC如果完全不设置storeClassName
   1. 如果存在默认的StoreClass，使用默认的StoreClass创建一个PV，并将其绑定至PVC
   2. 如果不存在默认的StoreClass，系统只选择未设定Class的PV与之匹配

## PVC筛选PV的流程
1. VolumeMode检查：筛选所有符合VolumeMode的Volume。Block/FileSystem
2. LabelSelector检查：若PVC配置了selector.matchLabels，会筛选符合条件的Volume
3. StorageClassName检查：如果配置了storageClassName，会筛选出value相同值的Volume
4. AccessMode检查：PV的accessMode需要与PVC的accessMode至少有一个相符合
5. Size检查：筛选出容量大于等于PVC声明容量的PV，如果数量大于1，则使用最小容量的PV

## PV状态流转
1. 创建
2. pending
3. available
4. bound
5. released
   1. deleted
   2. failed
> PV一旦到达released状态后，是无法根据ReclaimPolicy重新回到available状态而再次与新的PVC绑定。此时，如果想服用PV中的数据，只能进行以下两种操作之一：
> 1. 复用old PV中的存储信息，建立新PV
> 2. 直接复用PVC对象，即不unbound PVC和PV(Statefulset处理存储状态的原理)

## PVC状态流转
1. 创建
2. pending：没有PV与其绑定时
3. bound：与PV进行绑定时
4. lost：与其绑定的PV被删除时。如果有新PV再次绑定，将会重新进入bound状态，只能绑定与原pv名字一样的新pv