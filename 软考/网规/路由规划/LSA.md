# LSA
Link State Advertisement(链路状态通告)是用于描述邻接路由器、直连路由器、跨区域信息的一种协议。OSPF协议使用LSA来装载和传输链路状态信息

## LSA报文头部
OSPF除了Hello报文外，其他的OSPF报文都携带LSA信息。通过`LS Type + Link State ID + Advertising Router`来确定一个唯一的LSA报文，后续的报文均是对其进行的更新操作(序号递增)
1. LSA age：2B，LSA本地已存在时间，最大3600s
2. Options：1B
3. LS Type：1B，LSA报文类型
4. Link State ID(LS ID)：4B，不同的LS Type对应不同的Link State ID
5. Advertising Router：4B，通告该LSA的Router-id
6. LS sequence number：4B，LS序号
7. LS checksum：2B
8. Length：2B

## LSA报文类型
| 类型代码 | 描述/名称           | 产生者           | 泛洪范围                                | 用途                                                               | 对应的Link State Id        |
| -------- | ------------------- | ---------------- | --------------------------------------- | ------------------------------------------------------------------ | -------------------------- |
| Type1    | Router-LSA          | 每个路由器       | 描述区域内                              | 描述某区域内路由器端口链路状态的集合                               | 生成该条LSA的Router ID     |
| Type2    | Network-LSA         | DR               | 该网络所属区域                          | 包含了该网络上所有连接路由器的列表，描述广播型网络和NBMA网络       | 所描述网段上DR的端口IP地址 |
| Type3    | Network-Summary-LSA | ABR              | 该LSA产生的区域                         | 描述AS内部本区域外某一网段的路由信息                               | 所描述的目的网段的地址     |
| Type4    | ASBR-Summary-LSA    | ABR              | ABR所连接的区域内泛洪(ASBR所在区域除外) | 描述到某一ASBR的路由信息                                           | 所描述的ASBR的Router ID    |
| Type5    | AS-Extarnal-LSA     | ASBR             | 整个AS内部                              | 描述AS外部某一网段路由信息，该信息无论通告到任何区域，都不会被改变 | 所描述的目的网段的地址     |
| Type7    | NSSA外部LSA         | NSSA区域内的ASBR | 用于通告本区域连接的外部路由            |
> Type3和Type5不描述拓扑信息，仅包含路由信息  
> Type3在经过其余ABR时，会重新生成Type3  
> Type7报文在经过NSSA的ARB时，会被转为Type5

## LSDB
由一组LSA形成的集合，称为LSDB

## OSPF区域与LAS报文类型的关系
| 区域类型               | 1&2  | 3      | 4&5    | 7      |
| ---------------------- | ---- | ------ | ------ | ------ |
| 骨干区域(区域0)        | 允许 | 允许   | 允许   | 不允许 |
| 非骨干区域，非末梢区域 | 允许 | 允许   | 允许   | 不允许 |
| 末梢区域               | 允许 | 允许   | 不允许 | 不允许 |
| 完全末梢区域           | 允许 | 不允许 | 不允许 | 不允许 |
| NSSA                   | 允许 | 允许   | 不允许 | 允许   |

## Type1报文体
Type1报文体由多个Link组成，Link描述了某路由所有的接口信息，信息结构：
1. Link ID：由Link种类决定
2. Link Data：由Link种类决定
3. Link Type：Link的种类
4. Metric：度量值

## Link的种类(与OSPF网络类型是两个概念)
| 种类     | 说明                                              | Link ID                | Link Data                      |
| -------- | ------------------------------------------------- | ---------------------- | ------------------------------ |
| P2P      | 本路由器到邻居路由器为点到点连接                  | 邻居的Router ID        | 该网段上本地接口的地址         |
| TransNet | 本路由器到邻居路由器是一个Transit型网络(广播网络) | DR的接口地址           | 该网段上本地接口的地址         |
| StubNet  | 本路由器到邻居路由器是一个Stub型网络(如回环网卡)  | 该Stub网段的IP网络地址 | 该Stub网段的掩码               |
| virtual  | 本路由器到邻居路由器是一个虚拟连接                | 虚连接邻居的Router ID  | 去往该虚连接邻居的本地接口地址 |

## Type2报文体
Type2报文体由一个掩码和多个Router-id组成
1. Netmask：DR端口所对应的掩码
2. Attached Router：DR网段中所有路由器的Router-ID

## Type3报文体
Type3报文体用于传递路由信息，而不是链路状态信息。由一个掩码和一个开销值组成。在经过ABR时会重新产生Type3，修改Adv rtr和metric值
1. Netmask：目的网段的掩码，目的网段在头部LSID中
2. Metric：从该LSA产生者到目的网段的开销，该值在同一区域中传递时不进行改变
