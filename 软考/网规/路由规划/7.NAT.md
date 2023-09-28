# NAT协议
网络地址转换协议，当私有IP需要与公有IP进行通讯时，需要将私网IP转化为公网IP

## 分类
1. 静态NAT
2. 动态NAT
3. 端口多路复用(PAT)
4. Easy IP
5. NAT Server
> 1-4为基于源地址进行NAT，5为基于目的地址进行NAT

## 静态NAT
固定的一对一地址映射

## 动态NAT
Basic NAT，不固定的一对一地址映射，NAT目标从一组IP中动态获取其中一个未使用的

## 端口多路复用(PAT)
PAT NAT，多对一地址映射，NAT同时会分配一个源端口号，实现NAT地址服用

## Easy IP
某些情况下，无法提前得知被分配的公网IP，可以使用Easy IP，使用接口的IP进行NAT

## NAT Server
将公网地址映射到私有地址的，基于目的地址的NAT