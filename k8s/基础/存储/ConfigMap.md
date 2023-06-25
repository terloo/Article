# ConfigMap
许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息。ConfigMap API给我们提供了向容器内部注入配置信息的机制。ConfigMap可以保存单个属性，也可以保存整个配置文件JSON二进制大对象。

1. 创建
   1. 使用目录创建
      ```shell
      kubectl create configmap <configmap_name> --from-file=<dirpath> 
      ```
      > 指定目录下的所有文件都会被用在ConfigMap中创建一个键值对，键的值就是文件名，**值就是整个文件内容**
      > 
   2. 使用文件
      与使用目录创建一样，只是--from-file指定的是一个文件，--from-file这个参数可以对同一个cm多次使用，效果与指定目录添加目录中所有文件一样
   3. 使用字面值创建
      利用`--from-literal`参数传递配置信息，该参数与可以多次使用。
      ```shell
      kubectl create configmap <configmap_name> --from-literal=<key1>=<value1> --from-literal=<key2>=<value2>
      ```

2. Pod中使用ConfigMap
   cm:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cm1
     namespace: default
   data:
     special.how: very
     special.type: charm
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata: 
     name: cm2
   data:
     log_level: INFO
   ```
   1. 使用ConfigMap来代替环境变量
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: cm-pod1
      spec:
        containers:
        - name: nginx1
          image: nginx
          command: ['/bin/sh', '-c', 'env']
          env:
          - name: SPECIAL_KEY_LEVEL  # 环境变量名
            valueFrom:
              configMapKeyRef:
                name: cm1      # 读取指定configmap
                key: special.how  # 的指定键的值作为环境变量值
          - name: SPECIAL_KEY_TYPE  
            valueFrom:
              configMapKeyRef:
                name: cm1
                key: special.type
          envFrom:  # 或者用这种方式直接指定一个configMap中所有的键作为环境变量
            - configMapRef:
              name: cm2  
      ```
   2. 用ConfigMap设置命令行参数
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: cm-pod1
      spec:
        containers:
        - name: fooo
          image: busybox
          command: ['/bin/sh', '-c', 'echo $(SPECIAL_KEY_LEVEL  ) $(SPECIAL_KEY_TYPE)']
          env:
          - name: SPECIAL_KEY_LEVEL  # 环境变量名
            valueFrom:
              configMapKeyRef:
                name: cm1      # 读取指定configmap
                key: special.how  # 的指定键的值作为环境变量值
          - name: SPECIAL_KEY_TYPE  
            valueFrom:
              configMapKeyRef:
                name: cm1
                key: special.type
          envFrom:  # 或者用这种方式直接指定一个configMap中所有的键作为环境变量
          - configMapRef:
            name: cm2
      ```
   
   3. 通过数据卷使用ConfigMap
      在数据卷中使用ConfigMap，有不同的选项，最基本的就是将cm导入数据卷。在这个数据卷中，cm中的所有键都会生成文件，文件名为键名，文件内容为键值。
      ```yaml
      apiVersion: v1
      kind: Pod
      metadata: 
        name: vol-cm
      spec:
        containers:
        - name: nginx1
          image: nginx
          command: ['/bin/sh', '-c', 'cat /etc/config/special.how']
          volumeMounts: 
          - name: config-volume # 在容器中挂载这个容器卷
            mountPath: /etc/config  # 指定挂载的路径
        volumes:
        - name: config-volume
          configMap: # 从指定的cm中导入文件生成一个容器卷
            name: special-config
            items:   # 也可以选取挂载哪些key
            - key: configfile1.properties
              path: configfile1.properties # 指定挂载采用的文件名
        restartPolicy: Never
      ```

   4. ConfigMap的热更新
      使用ConfigMap挂载的Env是不会随着ConfigMap的更新而更新的，而使用ConfigMap挂载的容器卷是会随着ConfigMap更新进行热更新（有10s左右延迟）。需要注意的是，配置文件的更新并不会触发服务的滚动更新。
