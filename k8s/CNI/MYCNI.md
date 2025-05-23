# 实现一个基于路由模式的CNI插件

## 条件
集群的节点之间二层可达

## 配置文件
kubelet默认的读取配置文件的位置`/etc/cni/net.d/10-my-cni-demo.conf`
   ```json
   {
       "cniVersion": "0.1.1",
       "name": "my-cni-demo",
       "type": "my-cni-demo",
       "podcidr": "10.10.0.0/16"
       // "plugins": []
   }
   ```
   | 字段       | 说明                                                                        |
   | ---------- | --------------------------------------------------------------------------- |
   | cniVersion | 版本                                                                        |
   | name       | cni名字                                                                     |
   | type       | cni插件可执行文件的名字                                                     |
   | podcidr    | 该节点pod的cidr范围，不同节点不能冲突，该配置一般是通过读取Node资源对象获取 |
   | plugins    | 拥有 |

## CNI插件可执行文件
1. kubelet默认的执行cni插件的位置`/opt/cni/bin/my-cni-demo`
   1. 调用时会将配置文件中的内容通过`/dev/stdin`传入
   2. 调用时会将容器相关的参数通过环境变量传入
      1. COMMAND: 执行的命令类型ADD、VERSION、DEL、GET
      2. IFNAME: 该容器的内置网卡名，ADD时传入。一般为eth0
      3. CONTAINERID: 该容器的id，ADD时传入。一般为hash值，如13d16b2a54e651ea142d034a968e2b620eca76006ea37da224c3d43a956e5237
      4. NETNS: kublet创建的好的容器的命名空间的位置，ADD时传入。一般为/proc/113532/ns/net
2. [bash实现的cni插件](./my-cni-demo)

## 同节点之间Pod的网络互通
同节点之间Pod网络可以直接通过节点中的虚拟网桥进行通讯

## 不同节点之间的Pod网络互通
需要在每个节点上配置到其他节点的路由
1. 将发往另一个节点的报文路由到目标节点的物理网卡上  
   `ip route add <目标节点的podcidr范围> via <目标节点的宿主机ip> dev <该节点的物理网卡>`

## Pod与外网之间的网络互通
在每个节点配置Pod到外网的SNAT
1. 如果是报文源地址是Pod，且不是通过虚拟网桥发出的。进行SNAT  
   `iptables -A POSTROUTING -t nat -s <该节点的podcidr范围> ! -o <该节点虚拟网桥ip> -j MASQUERADE`