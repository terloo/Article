# PodSecurityPolicy

**已在1.25版本废弃**，由`PodSecurityAdmission`取代  
在创建Pod时，校验Pod指定的`securityContext`是否满足规则，不满足则不允许创建。在AdmissionController中使用Plugin实现，已默认启用。

## Sample
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: demo-psp
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - '*'
```
