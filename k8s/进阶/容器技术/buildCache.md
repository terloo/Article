# buildCache
Docker读取Dockerfile中的指定构建镜像时，会先判断缓存中是否有可用的镜像缓存，只有缓存不存在时才会重新构建

1. 通常Docker简单判断dockerfile中的指令与镜像
2. 针对ADD和COPY指令，Docker判断镜像层每一个文件的内容并生成一个checksum，与现存镜像比较时，直接比较checksum
3. 其他指令，比如`RUN apt-get -y update`，docker简单判断与现存镜像的指定字符串是否一致
4. 当某一层cache失效后，所有层级的cache一并失效，后续指令都需要重新构建