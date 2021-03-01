---
title: 自建Quay.io代理及Containerd&Docker代理配置
date: 2021-03-01 11:07:36
tags:
- Docker
- Containerd
---

> 首先的首先，需要一个代理，要不后面的看了也没用

# Docker镜像下载加速

此时（2021年3月1日），服务器配了镜像代理地址也无法拉取镜像，输出如下：

```
Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

第一个想到就是被墙了，但是系统全局配了`http_proxy`和`https_proxy`，**无效**

参考官方文档[Docker systemd/#httphttps-proxy配置](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)，给Dokcer daemon配置代理即可。

## 配置

1. 创建`docker.service.d`目录

```
mkdir /etc/systemd/system/docker.service.d
```

2. 然后在该目录下创建`http-proxy.conf`文件，在其中填入全局变量即可

```
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80/"
Environment="HTTPS_PROXY=http://proxy.example.com:80/"
```

3. [可选]添加不走代理的地址

```
Environment="HTTP_PROXY=http://proxy.example.com:80/"
Environment="NO_PROXY=localhost,127.0.0.0/8,docker-registry.somecorporation.com"
```

4. reload daemon

```
$ sudo systemctl daemon-reload
```

5. [可选] 检查配置是否加载

```
$ sudo systemctl show --property Environment docker
Environment=HTTP_PROXY=http://proxy.example.com:80/
```

6. 重启Docker:

```
$ sudo systemctl restart docker
```

# Containerd镜像下载加速

> 以1.4.1版本为例

## 配置

修改默认配置文件 `/etc/contaienrd/config.toml`，按如下配置即可，可修改为自己所需的地址

```toml
...
[plugins]
  [plugins.cri]
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = [
            "https://docker.mirrors.ustc.edu.cn",
            "http://hub-mirror.c.163.com"
          ]
...
```

# Quay.io 内网代理

quay的镜像现在已经下不下来了，只能走代理下载，现有国内的几个代理地址（中科大，七牛等）好像都不太行，只能自己内网建一个quay的代理了。

## 配置

Docker的registry镜像支持代理模式，先启动一个redis

```
# 创建网卡
> docker network create docker-registry
# 启动redis
> docker run -d --name=redis --net=docker-registry --restart=always redis redis-server --maxmemory 512m
```

创建registry的镜像存储路径，创建配置

```yaml
> mkdir -p /opt/docker/dockerhub/data
> cat > /opt/docker/dockerhub/data/config.yml <<EOF
version: 0.1
log:
    level: error
storage:
    delete:
        enabled: true
    cache:
        blobdescriptor: redis
    filesystem:
        rootdirectory: /var/lib/registry
    maintenance:
        uploadpurging:
            enabled: false
http:
    addr: :5000
    debug:
        addr: localhost:5001
    headers:
        X-Content-Type-Options: [nosniff]
notifications:
    endpoints:
        - name: local-5003
          url: http://localhost:5003/callback
          headers:
              Authorization: [Bearer 123456789]
          timeout: 1s
          threshold: 10
          backoff: 1s
          disabled: true
        - name: local-8083
          url: http://localhost:8083/callback
          timeout: 1s
          threshold: 10
          backoff: 1s
          disabled: true
health:
    storagedriver:
        enabled: true
        interval: 10s
        threshold: 3

# 替换为需要代理的地址，还可以配置用户名密码
proxy:
    remoteurl: https://quay.io

redis:
    addr: redis:6379
EOF
```

启动 docker registry，**一定要配置代理的全局变量**

```
> docker run -d \
--name dockerhub-mirror --restart always \
--network docker-registry \
-v /opt/docker/dockerhub/data:/var/lib/registry \
-v /opt/docker/dockerhub/config.yml:/etc/docker/registry/config.yml:ro \
-p 5000:5000/tcp --log-driver journald --log-opt tag="dockerd-dockerhub" \
-e http_proxy="http://proxy.example.com:80/" \
-e https_proxy="http://proxy.example.com:80/" \
registry:2.7.1
```

然后我们在Docker的daemon.json和Containerd的config.toml中配置该registry地址即可。

## 运行时配置

下面以内网地址`10.20.89.33`为例

### Docker

/etc/docker/daemon.json

```
...
"insecure-registries": ["10.20.89.33:5000"]
....
```

### Containerd

/etc/contaienrd/config.toml

```
...
[plugins]
  [plugins.cri]
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        ... 其他代理配置
        [plugins.cri.registry.mirrors."quay.io"]
          endpoint = [
            "http://10.20.89.33:5000"
          ]
...
```

# 参考

[1] [Docker systemd/#httphttps-proxy配置](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)

[1] [Docker systemd/#httphttps-proxy配置](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)

[2] [ustc Docker Hub 源使用帮助](http://mirrors.ustc.edu.cn/help/dockerhub.html)

