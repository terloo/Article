# slice

## 底层结构
1. data：底层数组的起始地址(并不一定是数组开头)，未初始化时为nil
2. len：存储元素个数
3. cap：容量，底层数组的长度

## 扩容(1.18)
threshold=256 oldCap原始容量 cap预估容量(oldCap+追加元素数) newCap新容量 finalCap最终容量
1. oldCap*2 < cap --> newCap = cap
2. oldCap < threshold --> newCap = oldCap*2
3. newCap = oldCap + (oldCap + 3*threshold) / 4
4. 对newCap进行内存对齐微调
   1. 计算出使用的内存字节数n = newCap * 切片类型对应的字节数
   2. 查询sizeclass.go表中bytes/obj列，找到最相近的数m(一样接近则取大值)
   3. finalCap = m / 切片对应的字节数
