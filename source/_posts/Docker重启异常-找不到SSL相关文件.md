---
title: Docker重启异常 - 找不到SSL相关文件
date: 2019-09-05 14:54:56
tags:
- Docker
---

## 前言

今天想把一台docker虚机配置一下Log的大小和DNS等信息，在`daemon.json`配置好以后，重启报错，死活就是起不来了，配置如下：

```json
{
"registry-mirrors": [
    "https://o61gnsy8.mirror.aliyuncs.com",
	"https://dockerhub.azk8s.cn",
	"https://reg-mirror.qiniu.com"
	],
"log-driver":"json-file",
"log-opts": {"max-size":"100m", "max-file":"3"},
"dns": ["223.5.5.5", "223.6.6.6"]
}
```

## 问题复现

上面的都不重要，和本次问题无关。当我们重启Docker时，发现启动失败！

```bash
[root@docker04 docker]# systemctl start docker
Job for docker.service failed because the control process exited with error code. See "systemctl status docker.service" and "journalctl -xe" for details.
[root@docker04 docker]# systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/etc/systemd/system/docker.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/docker.service.d
           └─10-machine.conf
   Active: failed (Result: start-limit) since 四 2019-09-05 11:59:13 CST; 1min 6s ago
     Docs: https://docs.docker.com
 Main PID: 3894 (code=exited, status=1/FAILURE)

9月 05 11:59:11 docker04 systemd[1]: docker.service: main process exited, code=exited, status=1/FAILURE
9月 05 11:59:11 docker04 systemd[1]: Failed to start Docker Application Container Engine.
9月 05 11:59:11 docker04 systemd[1]: Unit docker.service entered failed state.
9月 05 11:59:11 docker04 systemd[1]: docker.service failed.
9月 05 11:59:13 docker04 systemd[1]: docker.service holdoff time over, scheduling restart.
9月 05 11:59:13 docker04 systemd[1]: Stopped Docker Application Container Engine.
9月 05 11:59:13 docker04 systemd[1]: start request repeated too quickly for docker.service
9月 05 11:59:13 docker04 systemd[1]: Failed to start Docker Application Container Engine.
9月 05 11:59:13 docker04 systemd[1]: Unit docker.service entered failed state.
9月 05 11:59:13 docker04 systemd[1]: docker.service failed.
```

这里能看到的问题就是`start request repeated too quickly for docker.service`，启动请求重复的太快，这关我屁事啊。再去看看推荐的`journalctl -xe`，下面的应该是一个完整的记录，但是也没有指出什么问题，还是之前的问题。

```
9月 05 14:39:36 docker04 systemd[1]: start request repeated too quickly for docker.service
9月 05 14:39:36 docker04 systemd[1]: Failed to start Docker Application Container Engine.
-- Subject: Unit docker.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit docker.service has failed.
-- 
-- The result is failed.
9月 05 14:39:36 docker04 systemd[1]: Unit docker.service entered failed state.
9月 05 14:39:36 docker04 systemd[1]: docker.service failed.
9月 05 14:40:01 docker04 systemd[1]: Started Session 28 of user root.
-- Subject: Unit session-28.scope has finished start-up
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
-- 
-- Unit session-28.scope has finished starting up.
-- 
-- The start-up result is done.
```

这就邪了门了，啥都没有啊，这提示拿到Google一搜，没有一个能解决我的问题的。

#### 第一次尝试：dockerd

但是有人指出，可以试试用`dockerd`，那就来试试吧

```
... 无关log ...
API listen on /var/run/docker.sock
```

它就会一直卡在这里，也过不去，尝试了好几次。有时候得强制退出才可以，这时候就得手动去清了下pid了。

#### 第二次尝试：系统日志

想着还是回到系统日志，看看有没有什么漏下的，又去查了下`journalctl `的用法，可以显示指定单元的日志

```bash
9月 05 14:39:34 docker04 systemd[1]: Stopped Docker Application Container Engine.
9月 05 14:39:34 docker04 systemd[1]: Starting Docker Application Container Engine...
9月 05 14:39:34 docker04 dockerd[4683]: time="2019-09-05T14:39:34.789689391+08:00" level=info msg="Starting up"
9月 05 14:39:34 docker04 dockerd[4683]: failed to create API server: could not read CA certificate "/etc/docker/ca.pem": open /etc/docker/ca.pem: no such file or directory
9月 05 14:39:34 docker04 systemd[1]: docker.service: main process exited, code=exited, status=1/FAILURE
9月 05 14:39:34 docker04 systemd[1]: Failed to start Docker Application Container Engine.
9月 05 14:39:34 docker04 systemd[1]: Unit docker.service entered failed state.
9月 05 14:39:34 docker04 systemd[1]: docker.service failed.
9月 05 14:39:36 docker04 systemd[1]: docker.service holdoff time over, scheduling restart.
9月 05 14:39:36 docker04 systemd[1]: Stopped Docker Application Container Engine.
9月 05 14:39:36 docker04 systemd[1]: start request repeated too quickly for docker.service
9月 05 14:39:36 docker04 systemd[1]: Failed to start Docker Application Container Engine.
9月 05 14:39:36 docker04 systemd[1]: Unit docker.service entered failed state.
9月 05 14:39:36 docker04 systemd[1]: docker.service failed.

```

欸！在这里发现了我们想要的信息：`failed to create API server: could not read CA certificate "/etc/docker/ca.pem": open /etc/docker/ca.pem: no such file or directory`

没有SSL的证书和密钥，但是查看了一下别的docker，在同级目录都没有相关文件啊。难道和这个无关？

## "死马当活马医"

#### 证书路径

突然想起来，之前在做集群的时候，用了三剑客之一的`docker-machine`，猜测可能是在做远程连接的时候使用的SSL，但是去管理主机一看，不但没有这些文件，连同级目录都没有！

现在起码知道了时因为没有SSL证书和密钥的问题，其实我又去查了如何绕过SSL启动Docker，查半天也没啥进展，还是接着查证书吧。

在[Missing server.pem file while booting up docker host on vsphere using machine](https://forums.docker.com/t/missing-server-pem-file-while-booting-up-docker-host-on-vsphere-using-machine/3757)一贴的回复中，看到了曙光！回复如下：

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/docker/docker-machine.png)

OK！立马去查看相应的地址，居然被删了！对应主机名的文件夹都没了....

但是其他的机子目录下还是有的，基本内容如下：

```
[root@manager docker03]# ll
总用量 32
-rw-r--r--. 1 root root 1029 8月   1 18:12 ca.pem
-rw-r--r--. 1 root root 1070 8月   1 18:12 cert.pem
-rw-------. 1 root root 2276 8月   1 18:12 config.json
-rw-------. 1 root root  411 8月   1 18:12 id_ed25519
-rw-------. 1 root root  101 8月   1 18:12 id_ed25519.pub
-rw-------. 1 root root 1679 8月   1 18:12 key.pem
-rw-------. 1 root root 1679 8月   1 18:12 server-key.pem
-rw-r--r--. 1 root root 1107 8月   1 18:12 server.pem
```

也不知道这个能不能用，死马当活马医吧，直接把所有pem文件都拷贝到docker04（问题机器）的`/etc/docker`目录下。

```
[root@docker04 docker]# systemctl reset-failed docker.service 
[root@docker04 docker]# systemctl start docker.service
```

启动成功！

#### 重新添加管理

重新创新docker04，轻松创建成功。

```bash
[root@manager docker03]# docker-machine create --driver generic --generic-ip-address=10.20.88.35 docker04
Running pre-create checks...
Creating machine...
(docker04) No SSH key specified. Assuming an existing key at the default location.
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with centos...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env docker04
```

## 问题分析

1. 针对系统日志的分析不足，当无法从表面的日志查询到有用信息时，需要深入查看具体单元日志。
2. 从大量垃圾信息中，查询到自己想到的信息，不是简简单单的🔍就可以的

🔚

------

