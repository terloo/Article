# playbook

## 核心元素
1. Hosts：执行任务的远程主机列表
2. Tasks：任务集
3. Variables：内置变量或者自定义变量
4. Templates：模板，可替换模板文件中的变量并实现一些简单的逻辑操作
5. Handlers，notify：由特定条件触发的操作，满足条件才可执行，否则不执行
6. Tags：标签，指定某条任务执行，用于选择运行playbook中的某些代码

## hosts
playbook中的每个play的目的都是为了让特定的主机以某个指定的用户身份执行某个任务。hosts用于指定要执行任务的主机，须事先定义在主机清单中
```
one.example.com
onw.example.com:two.example.com
192.168.1.50
192.168.1.*
webserver:dbserver   # 两个组的并集
webserver:&dbserver  # 两个组的交集
webserver:!appserver # 在webserver组且不在appserver组
```
```yaml
- hosts: webserver:dbserver
```

## remote_user
指定执行任务所使用的用户

## tasks
对一个模块调用称为一个task，各个task按顺序在hosts指定的主机上依次执行。每个task都应该指定其name，如果没有name，则结果将用于输出
```yaml
- hosts: webserver
  remote_user: root  # 默认为root
  tasks:
  - name: install httpd
    yum: name=httpd        # module_name: module_args
  - name: start httpd
    yum: name=httpd state=started enabled=yes
```

### handlers和notify
每个task执行完毕时，可以触发一个信号，称为notify。在playbook中，可以配置handlers，来配置指定信号触发时应执行的操作
```yaml
- hosts: webserver
  remote_user: root
  tasks:
  - name: install httpd
    yum: name=httpd
  - name: install config file
    copy: src=files/httpd.conf dest=/etc/httpd/conf/
    notify: restart httpd                            # 如果成功执行了该task，那么会触发一个名为restart httpd的notify
  - name: start httpd
    service: name=httpd state=started enabled=yes
  handlers:
  - name: restart httpd  # 触发的信号
    service: name=httpd state=restarted  # 触发信号后执行的动作
```

### tags
利用tags标签，可以为特定的task指定标签。在指定playbook时，可以选择性的执行特定的tags，而非整个playbook文件。由tags执行的task也会触发notify
```yaml
- hosts: webserver
  remote_user: root
  tasks:
  - name: install httpd
    yum: name=httpd state=present
  - name: install config file
    copy: src=files/httpd.conf dest=/etc/httpd/conf/
    tags: conf
  - name: start httpd
    service: name=httpd state=started enabled=yes
    tags: service
```
```sh
# 列出所有的标签
ansible-playbook --list-tags
# 利用标签来指定需要执行的task，只要有其中一个标签，即可执行
ansible-playbook -t conf,service playbook.yaml
```

## when
利用when，可以通过条件判断选择性的执行的执行某个task，jinja2语法。由tags执行的when也会触发notify
```yaml
- hosts: webserver
  remote_user: root
  tasks:
  - name: shutdown if redhat
    command: /sbin/shutdown -h now
    when: ansible_os_family == "RedHat"
```

## with_items
在需要将一个任务重复执行多次时，可以使用with_items定义多个元素。对元素的引用，固定变量名为`item`
```yaml
- hosts: webserver
  remote_user: root
  tasks:
  - name: add serveral users
    user: name={{ item }} state=present groups=wheel
    with_items:
    - user1
    - user2

---
- hosts: webserver
  remote_user: root
  tasks:
  - name: add serveral users
    user: name={{ item.name }} group={{ time.group }} state=present groups=wheel
    with_items:
    - name: user1
      group: group1
    - name: user2
      group: group2
```