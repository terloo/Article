# Volume
在kubernets中，与docker不同，一旦镜像崩溃，kubelet就会以一种最初的镜像状态重启pod。

1. 背景
   kubernets中的卷有明确的声明周期--与封装它的Pod相同。所以，Pod中的卷的生命周期是大于Pod中的容器的生命周期的，当Pod中容器重启时，数据仍然得以保存。当然，当Pod重启，卷也就不复存在。kubernets支持很多种类型的卷，Pod可以同时使用任意数量的卷。

2. 卷的类型
   `emptyDir`,`hostPath`

3. emptyDir
   当Pod被分配到节点时，首先创建emptyDir卷，并且只要Pod在该节点上运行，该卷就会存在。正如卷名，它最初是空的。Pod的容器可以读取和写入emptyDir卷中的相同文件，尽管该卷可以挂载到每个容器的相同或不同路径上。但当出于任何原因从节点中删除Pod时，empty中的数据将会被永久删除。
   > empty卷中的数据在容器崩溃而Pod完好时是安全的。
   empty的用法有：
   - 暂存空间，如用于基于磁盘的合并排序
   - 用于长时间计算崩溃时的检查点
   - web服务器容器提供数据时，保存内容管理器容器提取的文件
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: vol-emptydir
   spec: 
     containers:
     - name: nginx1
       image: nginx:1.7.9
       volumeMonut:
       - mountPath: /cache
         name: cache-vol
     - name: nginx2
       image: nginx:1.7.9
       volumeMount:
       - mountPath: /cache2
         name: cache-vol
     volumes:
       - name: cahce-vol
         emptyDir: {}
   ```

4. hostPath
   hostPath卷将主机节点的文件系统中的文件或目录挂载到集群中。
   1. hostPath的类型（type属性）
      
      | 值                 | 说明                                                                |
      | ----------------- | ----------------------------------------------------------------- |
      |                   | 空字符串（默认）用于向后兼容，这意味着在挂载hostPath卷之前不会执行任何检查                         |
      | DirectoryOrCreate | 如果在给定的路径上没有任何东西存在，那么将根据需要在那里创建一个空目录，权限设置为0755，与kubetel具有相同的组和所有权。 |
      | Directory         | 给定的路径下必须是目录                                                       |
      | FileOrCreate      | 如果给定给的路径下没有任何东西存在，那么将根据需要在那里创建一个新文件，权限设置为0655，与kubetel具有相同的组和所有权  |
      | File              | 给定的路径必须是文件                                                        |
      | Socket            | 给定的路径必须是UNIX套接字                                                   |
      | CharDevice        | 给定的路径必须是字符设备                                                      |
      | BlockDevice       | 给定的路径必须是块设备                                                       |
   
   2. 注意事项
      1. 由于每个节点上的文件都不同，具有相同配置的Pod在不同节点上运行行为可能不一致
      2. 当Kubernets按照计划添加资源感知调度时，将无法考虑hostPath使用的资源
      3. 在底层主机上创建的文件或目录只能由root写入。需要在特权容器中以root身份运行进程，或修改主机上的文件权限以方便写入hostPath卷
   
   3. 资源清单示例
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: vol-hostpath
      spec:
        containers:
        - name: nginx
          images: nginx:1.7.9
          volumeMounts:
          - name: hostpath-demo
            path: /etc/nginx/conf.d  # 容器目录
        volumes:
        - name: hostpath-demo
          hostPath:
            path: /etc/nginx/conf.d  # 主机目录
            type: File
      ```
