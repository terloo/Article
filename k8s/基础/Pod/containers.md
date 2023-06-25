# containers
containers是Pod中用于定义组成Pod所有容器的字段

## 基础
| 字段名                 | 字段类型 | 说明           |
| ---------------------- | -------- | -------------- |
| sepc.containers[]      | list     |                |
| spec.containers[].name | string   | 定义容器的名字 |

## 端口
ports字段用于定义container需要暴露出的容器端口
| 字段名                                  | 字段类型 | 说明                                                                               |
| --------------------------------------- | -------- | ---------------------------------------------------------------------------------- |
| sepc.containers[].ports[]               | list     |                                                                                    |
| spec.containers[].ports[].name          | string   | 指定端口名称                                                                       |
| spec.containers[].ports[].containerPort | string   | 指定容器需要监听的端口号                                                           |
| spec.containers[].ports[].hostPort      | string   | 指定容器主机需要监听的端口号，如果设置了这个值将无法在同一台宿主机上启动两个该容器 |
| spec.containers[].ports[].protocol      | string   | 指定端口协议，TCP or UDP，默认TCP                                                  |

## 环境变量
env字段用于定义container中的环境变量
| 字段名                        | 字段类型 | 说明         |
| ----------------------------- | -------- | ------------ |
| spec.containers[].env[]       | list     |              |
| spec.containers[].env[].name  | string   | 环境变量名称 |
| spec.containers[].env[].value | string   | 环境变量值   |

## 镜像
用于定义container使用的镜像和拉取策略
| 字段名                            | 字段类型 | 说明               |
| --------------------------------- | -------- | ------------------ |
| spec.containers[].image           | string   | 定义要用到的镜像名 |
| spec.containers[].imagePullPolicy | string   | 定义镜像的拉取策略 |
> 镜像拉取策略（默认Always）：
> 1. Always：每次都尝试重新拉取镜像
> 2. Never：仅使用本地镜像
> 3. IfNotPreset：如果没本地没有就尝试拉取镜像

## 容器命令
用于定义container的工作目录和运行命令
| 字段名                       | 字段类型 | 说明                                                               |
| ---------------------------- | -------- | ------------------------------------------------------------------ |
| spec.containers[].command[]  | list     | 指定容器的启动命令，可以指定多个，不指定则使用镜像打包时使用的命令 |
| spec.containers[].args[]     | list     | 指定容器启动命令参数，可以指定多个                                 |
| spec.containers[].workingDir | string   | 指定容器的工作目录                                                 |

## 容器卷挂载
volumeMounts用于定义挂载到container中的容器卷
| 字段名                                     | 字段类型 | 说明                        |
| ------------------------------------------ | -------- | --------------------------- |
| spec.containers[].volumeMounts[]           | list     |                             |
| spec.containers[].volumeMounts[].name      | string   | 指定被容器挂载的存储卷名称  |
| spec.containers[].volumeMounts[].mountPath | string   | 指定存储卷挂载路径          |
| spec.containers[].volumeMounts[].readOnly  | boolean  | 是否为只读模式，默认为false |

## 资源
resources字段用于定于container所使用的资源限制
| 字段名                                     | 字段类型 | 说明                                     |
| ------------------------------------------ | -------- | ---------------------------------------- |
| spec.containers[].resource                 | Obj      |                                          |
| spec.containers[].resource.limits          | Obj      | 指定容器运行的资源上限相关信息           |
| spec.containers[].resource.limits.cpu      | string   | 指定容器运行的cpu上限，单位为core数      |
| spec.containers[].resource.limits.memory   | string   | 指定容器运行的内存上限，单位为MIB、GIB   |
| spec.containers[].resource.requests        | Obj      | 指定容器运行前需设置的环境变量列表       |
| spec.containers[].resource.requests.cpu    | string   | 指定容器初始化时的cpu值，单位为core数    |
| spec.containers[].resource.requests.memory | string   | 指定容器初始化时的内存值，单位为MIB、GIB |
