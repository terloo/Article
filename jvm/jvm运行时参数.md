# jvm运行时参数

## jvm参数选项类型
1. 标准类型参数：稳定，不容易变更
2. -X参数: 较为稳定，运行java -X可以列出所有-X参数
   1. Xint: 禁用JIT，所有字节码被解释执行，这个模式速度最慢
   2. Xcomp: 所有字节码第一次使用就都被编译成本地代码，然后再执行
   3. Xmixed: 混合模式，默认模式，让JIT根据程序运行的情况，有选择性的将部分字节码编译成本地代码
   4. -Xmx -Xms -Xss是-XX的缩写
3. -XX参数: 非标准参数，不稳定

## 常用的JVM运行时参数

### 打印设置的XX选项及值
1. -XX:+PrintCommandLineFlags: 让程序在运行前打印用户手动设置或JVM自动设置的XX选项值
2. -XX:+PrintFlagsInitial：打印出所有XX参数的默认值
3. -XX:+PrintFlagsFinal: 打印出所有XX参数的值，:=表示改值为非默认值
4. -XX:+PrintJVMOptions: 打印JVM的参数

### 堆、栈、方法区等内存大小设置
1. 栈
   1. -Xss128k: 设置每个线程的栈的大小
2. 堆
   1. -Xms500m：等价于-XX:InitialHeapSize，设置JVM初始堆内存
   2. -Xmx1000m：等价于-XX:MaxHeapSize，设置JVM最大堆内存
   3. -Xmn2g：设置年轻代的大小，官方推荐为整个堆内存大小的3/8，相当于同时设置NewSize和MaxNewSize
   4. -XX:NewSize=：设置年轻代初始值
   5. -XX:MaxNewSize=：设置年轻代最大值
   6. -XX:SurvivorRatio=：设置年轻代中Eden区与一个Survivor区的比值，这个参数必须显示指定，默认值不生效
   7. -XX:+UseAdaptiveSizePolicy=：自动分配Eden和Survivor区的比例，SurvivorRatio显示设置时不生效
   8. -XX:NewRatio=: 设置老年代与年轻代的比值
   9. -XX:PretenureSize Threshold=: 设置让大于此阈值的对象直接分配在老年代，只对Serial、ParNew生效
   10. -XX:MaxTenuringThreshold=: 对象的年龄大于此值时进入老年代，默认值为15
   11. -XX:+PrintTenuringDistribution: 让JVM在每次MinorGC后打印出当前使用的Survivor中对象的年龄分布
   12. -XX:TargetSurvivorRatio: 表示MinorGC后Survivor区中占用空间的期望比例
3. 方法区
    1.  -XX:MetaSapceSize=:元空间初始大小
    2.  -XX:MaxMetaSapceSize=: 元空间最大大小
    3.  -XX:+UseCompressedOops=: 压缩对象指针
    4.  -XX:+UseCompressedClassPointers: 压缩类指针
    5.  -XX:CompressedClassSpaceSize=: 设置Class MetaSapce的大小，默认1G
4.  直接内存  -XX:MaxDirectMemorySize=: 指定DirectMemory容量，如果不指定，则默认与java堆最大值大小一致

### OutOfMemory相关选项
1. -XX:+HeapDumpOutOfMemoryError: 表示堆内存出现OOM时，将堆转存到文件以便后续分析
2. -XX:+HeapDumpBeforeFullGC: 表示出现FullFC之前，生成Heap转存文件
3. -XX:HeapDumpPath: heap转存文件的路径
4. -XX:OnOutOfMemoryError: 出现OutOfMemory错误时，执行指定的脚本
