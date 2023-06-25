# Helm
Helm是k8s资源清单打包管理器的事实标准

## 概念
1. Helm Chart是创建一个应用实例的必要的配置组，也就是一堆Spec
2. 配置信息被归类为模板(Template)和值(Value)，这些信息经过渲染生成最终的对象
3. 所有的配置可以被打包进一个可发布对象中
4. 一个release就是一个有特定配置的chart的实例

## 组件
1. Helm client
   1. 本地chart开发
   2. 管理repository
   3. 管理release
   4. 与helm library交互
      1. 发送需要安装的chart
      2. 请求升级或者卸载存在的release
2. Helm library：负责与APIServer交互，并提供以下功能
   1. 基于chart和configuration创建一个release
   2. 把chart安装进kubernetes，并提供相应的release对象
   3. 升级和卸载
   4. Helm使用kubernetes存储所有信息，无需自己的数据库