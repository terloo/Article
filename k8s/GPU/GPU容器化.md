# GPU容器化

## CUDA架构
1. Nvidia显卡驱动：安装在宿主机上，通过容器卷挂载到容器中
2. CUDA工具库：打包进容器镜像中，实现版本和环境隔离
3. 应用(Tensorflow,PyTorch)：打包到容器镜像中，实现版本和环境隔离

## docker启动GPU的示例
```sh
docker run -it 
    --volume=nvidia_driver_xxx.xx:/usr/local/nvidia:ro \ # 将驱动挂载到容器中
    --device=/dev/nvidiactl \
    --device=/dev/nvidia-uvm \
    --device=/dev/nvidia-uvm-tools \
    --device=/dev/nvidia0 \  # 挂载显卡设备
    nvidia/cuda nvidia-smi   # 使用cuda镜像，执行nvidia-smi命令
```

## 部署步骤
1. 安装nvidia驱动
   1. `sudo yum install -y gcc kernel-devel-$(uname -r)`
   2. `sudo /bin/sh ./NVIDIA-Linux-x86_64*.run`
2. 安装NVIDIA docker2，Nvidia会替换掉docker默认的runc
   1. `sudo yum install nvidia-docker2`
   2. `sudo pkill -SIGHUP dockerd`
3. 部署Nvidia Device Plugin，DevicePlugin会以DaemonSet的形式存在于k8s中
   1. `kubectl create -f nvidia-device-plugin.yaml`

