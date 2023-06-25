# Pod生命周期
1. pause：pause容器会被首先创建完成
2. Init C：Pod创建前首先要进行Init C（容器初始化）的动作，完成一些容器创建前的必须操作。Init C在Pod初始完成后就会死亡。Init C可以有零个或多个，每个MianC也可以拥有零个或多个InitC，但InitC无法并行执行
3. Main C：主容器，可以有多个
   1. postStart：在容器创建之后，立即执行一个操作
   2. preStop：在容器进入Terminating状态后与宽限期并行一个操作，如果宽限期结束后preStop还未执行完毕，则再进入2s的小宽限期。preStop执行完毕或宽限期到期后，才会继续执行Pod的删除流程。
4. 探针：在主容器运行或启动的过程中侦测主容器的状态
   1. startup：探测成功后，再进行其他探针的探测
   2. readiness：在容器创建后，判断容器内部的进程是否准备好对外提供服务。可以定义容器在启动多少秒后再进行readiness的操作。检测成功后Pod的状态会变为Running
   3. liveness：在容器创建运行时，判断容器内部的进程是否假死或无法对外访问，执行相应操作

## Pod的状态机制
| 状态     | 描述                                                                                          |
| -------- | --------------------------------------------------------------------------------------------- |
| Pending  | Pod已经被k8s所接受，但仍未调度完毕或容器的镜像仍未拉去完毕或pause网络环境未生成完毕           |
| Running  | Pod已经被调度到节点，所有容器也已创建，至少有一个容器在running，或者在staring或restarting状态 |
| Succeded | Pod的所有容器已经非0退出，并且将不会被重启                                                    |
| Failed   | Pod的所有容器已经终止，并且至少有一个容器非0退出或者被系统终止                                |
| Unknown  | 由于某种原因，该Pod无法在CRI中被观察到状态                                                    |

## Pod资源对象的状态位
| 字段                                 | 说明                                                 |
| ------------------------------------ | ---------------------------------------------------- |
| status.phase                         | Pod的聚合状态，由所有conditions计算出的Pod的总体状态 |
| status.conditions                    | Pod各个阶段的状态的细节                              |
| status.conditions.type               | 阶段名                                               |
| status.conditions.status             | 阶段状态                                             |
| status.conditions.lastProbeTime      | 上次探测时间                                         |
| status.conditions.lastTransitionTime | 上次状态转换时间                                     |
| status.containerStatuses             | Pod各个容器的状态                                    |
| status.containerStatuses.started     | 容器是否已经启动，liveness探针提供状态               |
| status.containerStatuses.ready       | 容器是否已经就绪，readiness探针提供状态              |
| status.containerStatuses.state       | 容器当前所处的状态                                   |

## 状态计算细节
| kubectl get po                | PodPhase | Conditions                                         |
| ----------------------------- | -------- | -------------------------------------------------- |
| Completed                     | Succeded | restartPolicy:Nerver and container exit with 0     |
| ContainerCreating             | Pending  |                                                    |
| CrashLoopBackOff              | Running  | Container exits                                    |
| CreateContainerConfigError    | Pending  | configMap "xx" not found, Secret "xx" not found    |
| ErrImagePull/ImagePullBackOff | Pending  | Back-off image pull                                |
| Error                         | Failed   | restartPolicy:Nerver and container exit with not 0 |
| Evicted                       | Failed   | Evicted                                            |
| Init:0/1                      | Pending  | init container don't exit                          |
| Init:CrashBackOff/Init:Error  | Pending  | Init container crashed(exit with not 0)            |
| OOMKilled                     | Running  | Container are OOMKilled                            |
| StartError                    | Running  | Container can't be started                         |
| Unknown                       | Running  | Node NotReady                                      |
| OutOfCpu/OutOfMemory          | Failed   | Scheduled, but it can't pass kubelet admit         |