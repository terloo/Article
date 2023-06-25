# SecurityContext
Pod定义包含了一个安全上下文，用于描述允许它请求访问某个节点上特定的Linux用户(如root)、获得特权或访问主机网络，以及允许它在主机节点上不受约束地运行其他控件。  
Pod安全策略可以限制哪些用户或服务账户可以提供安全上下文设置。例如：Pod的安全策略可以限制卷挂载，尤其是hostpath，这些都是Pod应该控制的一些方面。  

## 配置SecrityContext的方法
- Container-level Security Context：仅应用到指定的容器
- Pod-level Security Context：应用到Pod内所有的容器以及Volume
- Pod Security Policies(PSP)：应用到集群内部所有Pod以及Volume
