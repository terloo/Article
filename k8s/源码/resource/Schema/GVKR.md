# GVKR
用于描述k8s中资源的组版本类和资源对象

## Group、Version、Resource、Kind相互之间组成的结构
存放在`vendor/k8s.io/apimachinery/pkg/runtime/schema/group_version.go`中，对一个或多个资源进行定位或描述时可能会使用到以下结构
| 结构名称             | 简称 | 说明                       |
| -------------------- | ---- | -------------------------- |
| GroupVersionResource | GVR  | 描述资源组、版本、资源     |
| GroupVersion         | GV   | 描述资源组、版本           |
| GroupResource        | GR   | 描述资源组、资源           |
| GroupVersionKind     | GVK  | 描述资源组、版本、资源种类 |
| GroupKind            | GK   | 描述资源组、资源种类       |
| GroupVersions        | GVS  | 描述资源组内多个版本       |
