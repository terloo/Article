# ReadinessGates
ReadinessGates属性给与了Pod以外的组件来控制Pod就绪状态的能力

## 清单
| 字段名              | 字段类型    | 说明           |
| ------------------- | ----------- | -------------- |
| spec.readinessGates | ArrayString | readinessGates |

## 功能
对于配置了readinessGates字段的Pod，k8s会在Pod的status.condition字段中寻找对应type的condition，在所有readinessGates中指定的条件均为True时，才认为Pod就绪  
如果找不到readinessGates中的条件，则该条件默认为False。如果第三方组件认为该Pod已经就绪，则可以通过apiServer将该Condition修改为True  
当Pod的容器准备就绪，但至少缺少一个自定义条件或False时，kubelet将Pod的条件设置为`ContainersReady`

```yaml
apiVersion: v1
kind: Pod
...
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready                              # a built in PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"        # an extra PodCondition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
...
```

## 实践
在mutating webhook中注入ReadinessGate。