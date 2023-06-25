# NFS搭建

1. 安装nfs工具
   `yum install nfs-utils -y`
2. 主节点执行
   1. `echo "/nfs/data *(insecure,rw,sync,no_root_squash)" > /etc/exports`
   2. `mkdir -p /nfs/data`
   3. `systemctl enable rpcbind --now`
   4. `systemctl enable nfs-server --now`
   5. `exportfs -r`使配置生效
3. 从节点执行
   1. `showmount -e master`查看服务是否启动
   2. `mkdir -p /nfs/data`
   3. `mount -t nfs master:/nfs/data /nfs/data`