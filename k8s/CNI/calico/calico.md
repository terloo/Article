# calico

## 主要工作组件
1. Felix：核心组件，运行在每个节点上。主要功能为接口管理、路由规则、ACL规则和状态报告。Felix会监听ETCD，从中获取节点创建，Pod创建等事件。用户创建Pod后，Felix负责初始化容器网络，并配置宿主机路由规则。如果用户指定了隔离策略，Felix也会将隔离策略创建到ACL中。
2. etcd：可与k8s公用，主要负责维护网络元数据信息
3. BGP Client(BIRD)：每个节点均有一个BIRD，负责读取Felix的路由信息读入内核，并通过BGP协议在集群中分发
4. BGP Router Reflector(RR)：为了避免BIRD形成全网mesh导致规模受限，所有的BIRD可以仅与RR相连接，减少网络连接数
5. calicoctl：calico命令行管理工具
