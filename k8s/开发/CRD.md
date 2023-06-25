# CRD

## 概念
复用k8s的apiServer，无需编写额外的apiServer。用户只需要自定义一个CRD，并且提供一个CRD控制器，就能通过k8s的api管理自定义资源对象，同时要求CRD对象符合apiServer的管理规范。

## 创建CRD的定义
1. 以istio的VirtualService为例子
```yml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 名字必须与spec字段匹配，格式为<名称复数形式>.<组名>
  name: demos.example.com
spec:
  group: example.com
  names:
    # 名称的复数形式，用于url
    plural: demos
    # 名称的单数形式，用于命令行和显示时的名字
    singular: demo
    # Kind名 通常是单数形式下的帕斯卡编码(PascalCased)形式，用于资源清单
    kind: Demo
    # shortName 简称，用于命令行
    shortName: 
    - dm
    # categories该资源的分类，在kubelet中可以直接获取该分类下所有资源对象
    categories:
    - istio-io
    - networking-istio-io
  # scope可以是Namespaced或者Cluster
  scope: Namespaced
  # 此CRD的所有版本
  versions:
  - name: v1
    # 该版本是否开启
    served: true
    # 有且只有一个版本能被标记为存储版本，用于ETCD存储
    storage: true
    # 定义资源的spec字段
    schema:
      # 使用的schema结构版本
      openAPIV3Schema:
        type: object
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          spec:
            type: object
            properties: 
              # 一个名为name的字段，类型是string
              name: 
                # 字段的类型
                type: string
                # 合法性校验，optional
                pattern: '^test.*$'
                # 默认值
                default: "testdemo"
    # 命令行获取时需要额外打印的字段
    additionalPrinterColumns:
    - name: 'Name of Demo'
      type: string
      description: The name of resource
      jsonPath: .spec.name
    # 子资源，目前支持status 和 scale
    subresources:
      # 启用status
      status: {}
      # 启用scale
      scale: 
        # scale对应的字段路径
        specReplicasPath: .spec.replicas
```
2. 字段解释
   1. group：设置api所属的组，将其映射为/apis/的下一级目录，此例子api的url为/apis/networking.istio.io
   2. scope：该api的生效范围，可选项为Namespaced、Cluster，默认值为Namespaced
   3. versions：设置此CRD支持的版本，可以设置多个版本。
      1. name：版本的名称
      2. served：是否启用
      3. storage：是否存储，只能有一个版本被设置为true
   4. names：CRD的名称，包括单数、复数、kind、所属组
      1. kind：