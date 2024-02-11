# ssh隧道
本地主机A，存在主机C无法直接被ssh连接，但有一台主机B可以ssh连接主机C

## 若A可以ssh连接B
1. 在主机A上执行命令，在主机A上建立本地隧道
```sh
ssh -L <A_tun_port>:<C_ssh_ip>:<C_ssh_port> -p <B_ssh_port> <B_username>@<B_ssh_ip>
```
   - A_tun_port：主机A上ssh隧道的监听端口
   - C_ssh_ip：主机C的ip地址
   - C_ssh_port：主机C的ssh端口
   - B_ssh_ip：主机B的ip地址
   - B_ssh_port：主机B的ssh端口
2. 在主机A上通过ssh隧道直接连接主机C
```sh
ssh -p <A_tun_port> <C_username>@<A_server_ip>
```
   - A_tun_port：主机A上隧道的端口
   - A_server_ip：主机A的地址
   - C_username：C的用户名

> 需要注意-L参数的完整格式为`-L [收听地址:]收听端口:目标主机:目标端口`，其中收听地址默认为`0.0.0.0`，代表隧道的监听地址为所有地址

## 若B可以ssh连接A
1. 在主机B执行命令，在主机A上建立远程隧道
```sh
ssh -R <A_tun_port>:<C_ssh_ip>:<C_ssh_port> -p <A_ssh_port> <A_username>@<A_ssh_ip>
```
2. 在主机A上通过ssh隧道直接连接主机C
```sh
ssh -p <A_tun_port> <C_username>@<A_server_ip>
```

> 需要注意-R参数的完整格式为`-L [收听地址:]收听端口:目标主机:目标端口`，其中收听地址默认为`0.0.0.0`，代表隧道的监听地址为所有地址


## 动态端口转发
动态端口转发实际上是一种特殊的本地隧道，可以把主机A上运行的ssh客户端变为SOCK5代理服务器，动态转发不固定目标地址和目标端口，而是读取应用程序的访问地址和访问端口
```sh
ssh -D <A_tun_port> -p <B_ssh_port> <B_username>@<B_ssh_ip>
```
在进行动态端口转发时，需要系统或应用实现代理功能，将所有流量发往本地`A_tun_port`端口

## 额外参数
1. `-N`：以不登录的形式建立连接
2. `-C`：压缩传输
3. `-f`：将ssh传输转入后台执行
4. 