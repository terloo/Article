# cri-dockerd
在k8s内置的docker-shim被移除之后，可以使用cri-dockerd来实现cri-api到docker的沟通

## 安装go语言环境

## 下载并编译cri-dockerd源码
```sh
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd && go build -o /usr/bin
cp -a packaging/systemd/* /etc/systemd/system
```

## 修改/etc/systemd/system/cri-docker.service
```
ExecStart=/root/go/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 --network-plugin=
```

## 修改`vim /etc/sysconfig/kubelet`
```
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd --container-runtime-endpoint=/var/run/cri-dockerd.sock"
```

## 开始服务
```
systemctl daemon-reload
systemctl enable --now cri-docker.service
systemctl enable --now cri-docker.socket
systemctl restart kubelet
```
