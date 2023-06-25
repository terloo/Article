# QoS
QoS(资源服务质量管理)是k8s中用于保证高可靠性Pod申请到高可靠资源，而非可靠Pod只能申请到非高可靠资源的一种机制，保证了资源超用系统的稳定

## 资源服务质量管理 Resource QoS
在超用(Over Committed，即容器的limits总和大于系统容量使用上限)系统中，容器负载的波动可能导致操作系统的资源不足，最终导致部分容器被杀掉。  
k8s中使用Qos来衡量容器被驱逐的优先级，即QoS等级，优先级从低到高
1. Guaranteed：完全可靠
2. Burstable：弹性波动、较可靠的
3. BestEffort：尽力而为、不太可靠的

## Qos计算方法
1. Guarateed：Pod的所有容器都定义了requests和limits(**包含自动填充**)，并且所有容器的limits和requests值相同(且不为0)
2. BestEffort：Pod中的所有容器都未定义Request和limits
3. Burstable：Pod即不是Guarateed，也不是BestEffort时，包含一下两种情况
   1. Pod中有容器定义了requests和limits，并且limits > requests
   2. Pod中有容器未定义requests和limits
4. **所有容器包含了主容器和初始化容器**