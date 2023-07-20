# role
role是ansible中用于层次性的组织playbook的功能。通过将不同的文件防止在不同的目录下，role能够自动加载对应的变量文件、tasks、handlers以及模板文件，在使用时只需要在playbook中使用include指令即可

## roles
roles是一系列role的集合，role只需要作为子目录放置在roles目录下即可
```
roles
|-role1
|-role2
|-role3
...
```
