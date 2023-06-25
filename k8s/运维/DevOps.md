# DevOps
DevOps主要由CI、Release Mgmt、CD组成

## CI
持续集成，在源代码修改完毕后，经过一系列流程将其合并至主分支的过程
1. Pull Request：向主分支提出合并请求
2. 可并行主要流程
   1. Build：编译
   2. Unit Test：单元测试
   3. E2E Test：用户行为测试
   4. Code Review：代码人工审查
3. Merge：合并到主分支

## Release Mgmt
代码合并进主分支后，将Commit cherry-pick到各Release Branch

## CD
持续部署，将修改应用到生产环境的过程
1. Offcial Build：编译、并构建生产环境镜像
2. E2E Conformance Test：整个系统的一致性测试
3. [Upgrading Test]：升级测试，用于版本升级行为测试
4. 各环境部署
   1. Soak：非正式环境
   2. Pre-prod：预生产环境
   3. Prod：生产环境

## 基于k8s的持续集成
将Jenkins部署在k8s中，Jenkins可以在构建时创建k8s的Job资源进行流水线操作