# unionfs
docker使用overlayFS作为unionfs的实现

## 特性
1. 将不同的目录挂载到同一个虚拟文件系统下的文件系统
2. 支持为每一个成员目录(branch)设定readonly、readwrite和whiteout-able权限
3. 文件系统分层，对readonly权限的branch可以逻辑上进行修改(增量修改)
4. 可以将多个disk挂载到同一个目录下
5. 可以将一个readonly的branch和writeable的brach联合在一起

## overlayFS
overlayFS是docker的默认文件驱动，已经被整合到kernel中。overlayFS分为两层，upper lower
1. lower层：即镜像层，lower层都是readonly权限，lower层可以有多个
2. upper：即容器层，容器启动后，将会在lower层之上，再附加一个readwrite权限的容器层
3. 合并层：upper层叠加到lower层之后形成的合并层。upper层和lower层同时存在同一文件时，将会使用upper的文件

## 示例
```shell
mkdir upper lower merged work
echo "from lower" > lower/in_lower.txt
echo "from upper" > upper/in_upper.txt
echo "from lower" > lower/in_both.txt
echo "from upper" > upper/in_both.txt
# 使用overlay的类型将lower目录和upper目录挂载到merged目录中，lowerdir可以用冒号挂载多个lower
# work作为中间临时文件夹，不能有任何内容，并且要与upper处于同一个文件系统
mount -t overlay overlay -o lowerdir=`pwd`/lower,upperdir=`pwd`/upper,workdir=`pwd`/work `pwd`/merged
```

## 文件操作
1. 增：增加文件是直接写到了upper文件中
2. 删：删除upper层中的文件将会直接删除，删除lower层的文件将会在upper层中形成一个遮罩文件阻止lower层的文件挂载到merge层中
3. 改：修改upper层中的文件将会直接修改，修改upper层的文件会将lower层的文件复制到upper中再进行修改
4. 读：读取upper和lower层中都存在的文件会读取upper层中的文件