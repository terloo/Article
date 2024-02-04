# rest
rest客户端是k8s四种客户端中的基础客户端，`k8s.io/client-go/rest`编写了与rest有关的所有逻辑

## 组成
1. Config：即`restclient.Config`，用于实例化客户端对象的配置类
2. transport：用于进行http请求，由标准库包装而来
3. RESTClient：客户端对象，每个GV都对应一个客户端，UnversionedRESTClient没有对应的GV，用于获取非资源对象
4. Request：请求类，包装了RESTClient，使用链式方法发送HTTP请求
5. Result：Request对象执行Do方法后拿到的结果