---
title: Win10 基于 Hyper-V 安装 Minikube
abbrlink: 1ef3876c
date: 2020-05-06 17:23:39
tags:
---

> 若相关文件无法下载，可以参考：
>
> https://cloud.189.cn/t/qEnA73VB3aAr（访问码：ukv4）

# 安装Minikube

参考[https://yq.aliyun.com/articles/221687](https://yq.aliyun.com/articles/221687) 在Windows10上安装`minikube`

1. 开启Hyper-V

   在 控制面板 - 程序和功能 - 启用或关闭Windows功能 中勾上 Hyper-V，重启设备

2. 下载`kubectl`和`minikube`

   参考[install-kubectl-on-windows](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-windows)下载kubectl

   当前的Relsease版是1.9.2，[link](https://github.com/kubernetes/minikube/releases/tag/v1.9.2)。更改文件名为`minikube.exe`

   将上面两个文件放在同一个文件夹内，设置全局Path

3. Hyper-V创建虚拟交换机

    首先应该打开Hyper-V管理器创建一个外部虚拟交换机。

   ![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/minikube/minikube_switch_1.png)

   ![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/minikube/minikube_switch_2.png)

记住这里设置的交换机名称，后面创建minikube时需要使用。例如我图中的就是`MiniKubeSwitch`. 

4. 安装minikube

   ```
   minikube.exe start --image-mirror-country cn --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.8.0.iso --registry-mirror=https://o61gnsy8.mirror.aliyuncs.com --vm-driver="hyperv" --hyperv-virtual-switch="MiniKubeSwitch" --memory=4096
   ```

   输出：

   ```
   * Microsoft Windows 10 Pro 10.0.18362 Build 18362 上的 minikube v1.9.2
   * 根据用户配置使用 hyperv 驱动程序
   * 正在下载 VM boot image...
       > minikube-v1.8.0.iso: 173.56 MiB / 173.56 MiB [-] 100.00% 7.43 MiB p/s 23s
   * Starting control plane node m01 in cluster minikube
   * Downloading Kubernetes v1.18.0 preload ...
       > preloaded-images-k8s-v2-v1.18.0-docker-overlay2-amd64.tar.lz4: 542.91 MiB
   * Creating hyperv VM (CPUs=2, Memory=4096MB, Disk=20000MB) ...
   * 找到的网络选项：
     - NO_PROXY=192.168.99.100
     - no_proxy=192.168.99.100
   ! This VM is having trouble accessing https://k8s.gcr.io
   * To pull new external images, you may need to configure a proxy: https://minikube.sigs.k8s.io/docs/reference/networking/proxy/
   * 正在 Docker 19.03.6 中准备 Kubernetes v1.18.0…
     - env NO_PROXY=192.168.99.100
     - env NO_PROXY=192.168.99.100
       > kubeadm.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
       > kubelet.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
       > kubectl.sha256: 65 B / 65 B [--------------------------] 100.00% ? p/s 0s
       > kubeadm: 37.96 MiB / 37.96 MiB [-----------] 100.00% 386.80 KiB p/s 1m40s/s ETA 9m52s
       > kubectl: 41.98 MiB / 41.98 MiB [-----------] 100.00% 399.64 KiB p/s 1m48s
       > kubelet: 108.01 MiB / 108.01 MiB [---------] 100.00% 832.75 KiB p/s 2m13s
   * Enabling addons: default-storageclass, storage-provisioner
   * 完成！kubectl 已经配置至 "minikube"
   ```

# 测试Minikube

通过一个Kubernetes官方示例，启动Nginx来演示minikube的使用

## 下载Yaml配置

先下载几个yaml文件，可以直接从官方下载。也可以从我的网盘下载。

官方：

```
# 部署Nginx
https://k8s.io/examples/application/deployment.yaml
# 升级Nginx
https://k8s.io/examples/application/deployment-update.yaml
# 扩容Nginx
https://k8s.io/examples/application/deployment-scale.yaml
```

也可以用我上传的：

https://cloud.189.cn/t/qEnA73VB3aAr（访问码：ukv4）

## 部署

```
λ kubectl.exe apply -f deployment.yaml
deployment.apps/nginx-deployment created
```

deployment.yaml文件内容如下，部署脚本主要关注点就是Nginx的版本，后期会对`1.14.2`这个版本进行升级。

```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

## 升级

```
λ kubectl.exe apply -f deployment-update.yaml
deployment.apps/nginx-deployment configured
```

查看nginx-deployment的详情，可以看到StrategyType为RollingUpdate，属于升级操作，而且镜像版本号变为了`1.16.1`。

```
λ kubectl.exe describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Wed, 06 May 2020 11:08:55 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=nginx
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.16.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-7b45d69949 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  110s  deployment-controller  Scaled up replica set nginx-deployment-7b45d69949 to 1
  Normal  ScalingReplicaSet  63s   deployment-controller  Scaled down replica set nginx-deployment-6b474476c4 to 1
  Normal  ScalingReplicaSet  63s   deployment-controller  Scaled up replica set nginx-deployment-7b45d69949 to 2
  Normal  ScalingReplicaSet  53s   deployment-controller  Scaled down replica set nginx-deployment-6b474476c4 to 0
```

## 扩容

```
λ kubectl.exe apply -f deployment-scale.yaml
deployment.apps/nginx-deployment configured
```

查看nginx-deployment的详情，可以看到Replicas变为了4个。

```
λ kubectl.exe describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Wed, 06 May 2020 11:08:55 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 3
Selector:               app=nginx
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.14.2
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-6b474476c4 (4/4 replicas created)
Events:
  Type    Reason             Age                  From                   Message
  ----    ------             ----                 ----                   -------
  Normal  ScalingReplicaSet  6m22s                deployment-controller  Scaled up replica set nginx-deployment-7b45d69949 to 1
  Normal  ScalingReplicaSet  5m35s                deployment-controller  Scaled down replica set nginx-deployment-6b474476c4 to 1
  Normal  ScalingReplicaSet  5m35s                deployment-controller  Scaled up replica set nginx-deployment-7b45d69949 to 2
  Normal  ScalingReplicaSet  5m25s                deployment-controller  Scaled down replica set nginx-deployment-6b474476c4 to 0
  Normal  ScalingReplicaSet  52s                  deployment-controller  Scaled up replica set nginx-deployment-6b474476c4 to 1
  Normal  ScalingReplicaSet  52s                  deployment-controller  Scaled up replica set nginx-deployment-7b45d69949 to 4
  Normal  ScalingReplicaSet  51s                  deployment-controller  Scaled down replica set nginx-deployment-7b45d69949 to 3
  Normal  ScalingReplicaSet  50s (x2 over 3h17m)  deployment-controller  Scaled up replica set nginx-deployment-6b474476c4 to 2
  Normal  ScalingReplicaSet  35s                  deployment-controller  Scaled down replica set nginx-deployment-7b45d69949 to 2
  Normal  ScalingReplicaSet  17s (x4 over 34s)    deployment-controller  (combined from similar events): Scaled down replica set nginx-deployment-7b45d69949 to 0
```

🔚

------



