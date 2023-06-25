# Strategy策略
k8s系统中，每个资源都有自己的Strategy(策略)操作，每个资源的特殊需求都可以在自己的Strategy策略接口中实现。  
每个策略接口有其配套的BeforeXXX使用函数，由Store进行调用
