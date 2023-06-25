# kubectl插件

## 安装
1. 安装：kubectl插件是一个可独立执行的可执行文件，名称以`kubeclt-`开头，需要将其配置到PATH中
2. 发现插件：kubeclt提供一个命令`kubectl plugin list`，用于搜索PATH中有效的插件执行文件，任何`kubectl-`开头的文件如果不可执行，则会抛出警告
3. 限制：无法创建覆盖现有的kubectl插件，如`kubectl version`将始终执行`kubectl-version`命令。由于这个限制，也不可能使用插件将新的子命令添加到现有的kubectl命令中。
4. 执行：使用`kubectl xxx`来执行`kubectl-xx`插件。