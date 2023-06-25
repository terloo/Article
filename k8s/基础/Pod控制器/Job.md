# Job

## 资源清单示例
```yaml
apiVersion: batch/v1
  kind: Job
  metadata:
  name: Job-pi
spec:
  completions: 1  # Job结束需要成功运行的Pod个数，默认为1次
  parallelism: 1  # 并行运行的Pod个数，默认为1次
  activeDeadlineSeconds: 3  # 运行失败的Pod的重试最大时间，超过这个时间不会重启
  template:
    metadata:
    labels:
      app: job-example
    spec:
      containers:
      - name: job-pi
        image: prel
        command: ['perl', '-Mbignum=bpi', '-wle', 'print bpi(2000)']
      restartPolicy: Never
```
