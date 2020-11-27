---
title: 访问Docker容器响应'ERR_CONNECTION_RESET'的问题排查
tags:
  - Docker
abbrlink: 6e5db5aa
date: 2020-06-10 14:14:39
---

# 前言

接到反馈说刚才好好的单体Docker容器无法访问了。浏览器访问超时并显示`ERR_CONNECTION_RESET`。

# 问题排查

首先想到的就是服务挂了，但是从`docker ps -a`来看，容器运行良好，且都对外开放了相应的端口。

## 1. 排查端口

宿主机使用 `lo` 网卡访问响应端口，看下端口连通性。

```bash
> wget -O- 127.0.0.1:8500
[root@localhost ~]# wget -O- 127.0.0.1:8500
--2020-06-10 22:25:43--  http://127.0.0.1:8500/
正在连接 127.0.0.1:8500... 已连接。
已发出 HTTP 请求，正在等待回应... 读取文件头错误（Connection reset by peer）。
重试中。
...
```

首先确认端口是通的，但是被拒绝了，下面抓包看一下。启动抓包并重新发送 wget 请求

```bash
> tcpdump port 8500 -i lo -vvv -w /srv/out.cap
```

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/docker/capture-rst.png)

正常发送HTTP请求，收到了RST。所以浏览器才会显示`ERR_CONNECTION_RESET`

经查，端口没问题。

## 2. 排查容器

已经知道了宿主机访问会出现问题，那就看下在容器内访问能不能正常响应，进入一个和目标容器相同子网的容器。这里以busybox为例，子网使用的 `bridge` 模式，容器启动时选择相同的 Docker 网卡即可

```bash
> docker exec -it busybox sh
> wget -O- 192.168.24.151:8500
Connecting to 192.168.24.151:8500 (192.168.24.151:8500)
*** HTML 内容 ***
```

能正常响应页面，在相同子网内访问是没有问题的。

## 3. 排查网卡

从前两项排查可知，宿主机无法访问，Docker子网内可以访问。那就是网卡应该出现了问题。因为是所有容器都是跑在新建的网卡下面的，所以直接查看宿主机网卡信息

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/docker/duplicate-interface.png)

居然有两个相同的网卡，居然是因为网卡冲突了。上一个出入流量很高，后一个就很低。查看现在正在使用的网卡ID，是用的上面这个流量高的网卡，而且后一个网卡也没有看到对应的网络信息。

```bash
[root@localhost ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
edbd45482644        bridge              bridge              local
a0b6ef0cf177        host                host                local
715b7fddecd5        xxx-network         bridge              local
ef3d1b9a5ebc        none                null                local
```

删掉没用的那个网卡就可以了

```bash
> ip link delete br-fae0e0150543
```

# 总结

网卡冲突带来的连接重置，通过后期查询[解决网卡冲突](https://blog.csdn.net/kunkliu/article/details/79082886)得知，Linux内核默认是给ARP做了VIP的，所以两个网卡都可以访问，但是docker什么时候创建的网卡就不得而知了。🔚

------

