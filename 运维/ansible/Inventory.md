# Inventory
Inventory(主机清单)是ansible中用于管理所有主机分类列表的文件，可以将所有主机按组进行使用，默认存储路径`/etc/ansible/hosts`，可以通过`-i`参数配置

## 文件格式
```
[web]  // 分类名
172.1.2.1

[db]
172.2.2.1
172.2.2.2
172.2.2.3

[nfs]
172.3.2.1
```

## 子组
对多个/单个已经划分好的组，进行再次创建分组
```
[web]  // 分类名
172.1.2.1

[db]
172.2.2.1
172.2.2.2
172.2.2.3

[nfs]
172.3.2.1

// 将web db nfs 三个组合并为一个组 allserver
[allserver:children]
web
nfs
db
```

## ssh配置
推荐使用ssh config而不是在主机清单进行配置
```
[web]
172.1.2.1 ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass='xxxxxx'

[db]
172.2.2.1
172.2.2.2
172.2.2.3

// 同组使用同一个ssh配置
[web:vars]
ansible_ssh_port=22
ansible_ssh_user=root
ansible_ssh_pass='xxxxxx'

// 所有服务器使用同一个配置
[all:vars]
ansible_ssh_port=22
ansible_ssh_user=root
ansible_ssh_pass='xxxxxx'
```