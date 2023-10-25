# code-generator
code-generator是k8s提供的一系列工具，用于对定义好的k8s资源对象生成
1. client-gen：clientSet客户端
2. lister-gen：lister客户端
3. informer-gen：informer客户端
4. deepcopy-gen：深拷贝方法

## 使用方法
1. 下载源码`https://github.com/kubernetes/code-generator.git`
2. 安装`go install ./cmd/{client-gen,deepcopy-gen,lister-gen,informer-gen}`
3. 使用源码中提供的脚本，批量生成代码(在待生成代码的go.mod目录下执行)`generate-groups.sh all <output-package> <apis-package> <groups-versions> <工具的命令行参数>`
   1. output-package：输出文件的包路径，如`github.com/example/project/pkg/generated`
   2. apis-package：外部版本api的包路径，如`github.com/example/project/pkg/apis`
   3. groups-versions：资源的Group和Version，如`groupA:v1,v2 groupB:v1 groupC:v2`
   4. 工具的命令行参数：generator系列工具的参数
      1. 需要指定协议模板文件`-h header.txt`，header.txt可以空文件，也可以是开源协议信息
      2. 需要指定生成文件的基础路径`-o ./`(默认值是./)

## 标记

### deepcopy 标记在类上
1. 关闭：`//+k8s:deepcopy-gen=false`
2. 打开：`//+k8s:deepcopy-gen=true`
3. 基于接口生成deepcopy方法：`//+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`

### clientset,lister,informer 标记在类上
1. 默认：`//+genclient`
2. 不操作子类型：`//+genclient:noStatus`
3. cluster级别：`//+genclient:nonNamespaced`
4. 无操作：`//+genclient:noVerbs`
5. 指定操作：`//+genclient:onlyVerbs=create,delete`
6. 指定不操作：`//+genclient:skipVerbs=get,list,deleteCollection,watch`

## deepcopy 标记在doc.go上
```go
// +k8s.io:deepcopy-gen=packge
// +groupName=foo.example.com
package v1
```