# role
role是ansible中用于层次性的组织playbook的功能。通过将不同的文件防止在不同的目录下，role能够自动加载对应的变量文件、tasks、handlers以及模板文件，在使用时只需要在playbook中使用include指令即可

## roles
roles是一系列role的集合，role只需要作为子目录放置在roles目录下即可
```
playbook.yaml
roles/
    role1/
    role2/
    role3/
...
```

## roles路径
1. /usr/share/ansible/roles
2. /etc/ansible/roles
3. ~/.ansible/roles

## role目录结构
```
roles/
  <role_name>/
    tasks/
    files/
    vars/
    templates/
    handlers/
    default/
    meta/
```
1. role_name：角色名
   1. files：存放由copy或者script等模块使用的文件
   2. templates：存放模板文件
   3. vars：存放变量文件。至少应该包含一个名为main.yaml的文件；目录下其他的文件需要在此文件中通过include进行引用
   4. tasks：定义task(role的基本元素)，一般情况下，一个task文件包含一个task。至少应该包含一个名为main.yaml的文件；目录下其他的文件需要在此文件中通过include进行引用
   5. handlers：定义handler。至少应该包含一个名为main.yaml的文件；目录下其他的文件需要在此文件中通过include进行引用
   6. meta：定义当前角色的特殊设定及其依赖关系，至少应该包含一个名为main.yaml的文件；目录下其他的文件需要在此文件中通过include进行引用
   7. default：设定默认变量时使用此目录的main.yaml文件，比vars优先级低

## playbook调用role
```yaml
- hosts: all
  remote_user: root
  roles:
  - role1 # 直接调用role
  - {role: role2, var1: value1, var2: value2}  # 调用nginx的role，并传递变量
  - {role: role3, var1: value1, var2: value2, when: <expr>}  # 基于条件，来决定role是否执行
```

## role中使用tags
```yaml
# playbook.yaml
- hosts: all
  remote_user: root
  roles:
  - role: role1
    tags:
    - tag1
    - tag2
  - role: role2
    tags:
    - tag2
    - tag3
  - role: role3
    tags:
    - tag1
    - tag3
```
```sh
# 利用标签来指定需要执行的task，只要有其中一个标签，即可执行
ansible-playbook -t "tag1,tag2,tag3" playbook.yaml
```