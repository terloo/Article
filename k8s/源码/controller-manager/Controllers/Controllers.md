# Controllers

## 所有的Controller及其作用
1. DeploymentController
2. JobController
3. NamespaceController：保证Namespace被删除时，先删除该Namespace下的所有资源
4. GarbageController：处理级联删除
5. NodeLifeCycleController：在Node的Taint和Pod的tolaretion发生变更时，2判断是否应该驱逐Pod，并根据tolaretion来确定驱逐时间