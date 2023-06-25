# Serializer
Serializer序列化时，资源的原始版本优先级更高

## 种类
在进行编解码操作时，每一种序列化器都资源对象的metav1.TypeMeta(即APIVersion和Kind字段)进行验证，如果Object未提供这些字段，就会返回错误
1. jsonSerializer：JSON格式的序列化/反序列化器。使用`application/json`的Content-Type作为标记
2. yamlSerializer：YAML格式的序列化/反序列化器。使用`application/yaml`的Content-Type作为标记
3. protobufSerializer：protobuf格式的序列化/反序列化器。使用`application/vnd.kubernetes.protobuf`的Content-Type作为标记
