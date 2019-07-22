---
title: Docker集群搭建及网络互通配置
tags:
  - docker
  - swarm
  - cluster
  - consul
abbrlink: '187e5652'
date: 2019-06-18 10:33:57
---

## 目的

现在手头有两个虚机，都内建了docker，但是在搭建[consul](https://www.consul.io/)的时候想试试其多dc的特性，所以就得保证两个docker能互相访问。

## 创建docker集群

默认已经分别安装docker，现在docker内置有`swarm`，直接使用就可以

两台虚机配置如下：

|         | VM-1        | VM-2         |
| ------- | ----------- | ------------ |
| ip      | 10.20.30.97 | 10.20.90.104 |
| 防火墙  | 关闭        | 关闭         |
| selinux | 关闭        | 关闭         |

docker版本：

```
Client:
 Version:           18.09.5
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        e8ff056
 Built:             Thu Apr 11 04:43:34 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.5
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.8
  Git commit:       e8ff056
  Built:            Thu Apr 11 04:13:40 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

### 初始化swarm

在`VM-1`上初始化，默认是manager节点

```bash
[root@bogon consul]# docker swarm init
Swarm initialized: current node (ggszd5frpg8wt7vfovh229xun) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3ww9xfy1w8opdd5rcn0a3s4ye3s4evnllyki9kne7oo1dpi2ia-4z7mvmuni39wp2bya7u4cynt8 10.20.90.97:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

然后将上面那个命令粘到`VM-2`执行

```bash
[root@bogon consul]# docker swarm join --token SWMTKN-1-3ww9xfy1w8opdd5rcn0a3s4ye3s4evnllyki9kne7oo1dpi2ia-4z7mvmuni39wp2bya7u4cynt8 10.20.90.97:2377
This node joined a swarm as a worker.
```

添加成功后，就可以看到节点信息

```bash
[root@bogon consul]# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
ggszd5frpg8wt7vfovh229xun *   bogon               Ready               Active              Leader              18.09.5
sh6tw0eyx9lbei1y8d1vbetps     bogon               Ready               Active                                  18.09.6

```

### 创建overlay

到manager节点上创建attachable的overlay network，名字叫做prod-overlay，同时可以检查网络列表

```bash
[root@bogon consul]# docker network create -d overlay --attachable prod-overlay
8pa6ndbius26x0j9u9m1sfldw
[root@bogon consul]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
bd80de1917f8        bridge              bridge              local
4a740c45a02b        docker_gwbridge     bridge              local
b156a97d7d2d        host                host                local
8pa6ndbius26        prod-overlay        overlay             swarm

```

此时在`VM-2`上是看不到这个网络的，执行完后面的命令会自动添加（？生成）进去

在`VM-1`上创建容器`testc1`，挂到`prod-overlay` network上：

```bash
[root@bogon consul]# docker run --name testc1 --network prod-overlay -itd busybox
```

在`VM-2`上创建容器`testc2`，挂到`prod-overlay` network上：

```bash
[root@bogon consul]# docker run --name testc2 --network prod-overlay -itd busybox
```

### 访问验证

查看`VM-2`docker的network，现在应该可以查看到了

```bash
[root@bogon consul]# docker network ls
NETWORK ID          NAME                     DRIVER              SCOPE
0c968a179326        bridge                   bridge              local
26c07c4bd000        host                     host                local
y6kdngxun2a3        ingress                  overlay             swarm
8pa6ndbius26        prod-overlay             overlay             swarm
```

### 互ping测试

`VM-1`ping`VM-2`

```bash
[root@bogon consul]# docker exec testc1 ping -c 2 testc2
PING testc2 (10.0.0.5): 56 data bytes
64 bytes from 10.0.0.5: seq=0 ttl=64 time=0.391 ms
64 bytes from 10.0.0.5: seq=1 ttl=64 time=0.620 ms

--- testc2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.391/0.505/0.620 ms
```

`VM-2`ping`VM-1`

```bash
[root@bogon consul]# docker exec testc2 ping -c 2 testc1
PING testc1 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.402 ms
64 bytes from 10.0.0.2: seq=1 ttl=64 time=0.363 ms

--- testc1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.363/0.382/0.402 ms
```



## 参考资料

1. [一种生产环境Docker Overlay Network的配置方案](https://chanjarster.github.io/post/docker-overlay-network/)
2. [docker swarm 和compose部署服务，解决跨主机网路问题和ip不固定问题（一）](http://blog.sina.com.cn/s/blog_ad5322e70102x1ex.html)

结束！🔚

------

