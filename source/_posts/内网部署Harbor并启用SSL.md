---
title: 内网部署Harbor并启用SSL
tags:
  - Harbor
  - Docker
  - SSL
abbrlink: 1d848727
date: 2019-07-22 14:50:45
---

## 申请SSL证书

> Let's Encrypt记录了很多的[ACME客户端实现](https://letsencrypt.org/docs/client-options/)，网上搜到的很多帖子都是用的官方推荐的`Certbot`,我在这里使用的`acme.sh`

基本的安装和申请查看官方文档就可以。

地址：[🔗安装&申请证书🔗](<https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E>)

因为我是在阿里云买的域名，所以直接使用DNS模式，申请一个泛域名证书，可以看这个 -> [🔗阿里云DNS自动申请🔗](<https://github.com/Neilpang/acme.sh/wiki/dnsapi#11-use-aliyun-domain-api-to-automatically-issue-cert>)

原文给出的获取access key链接失效，会跳转到RAM（权限控制）页面，也推荐使用子账户的access key去做这个操作。

1. 登录RAM：[地址](<https://ram.console.aliyun.com/overview>)

2. 创建一个新的用户，获取Access Key

   新建用户 -> 填入相关信息，勾选**编程访问** -> 保存Access Key（CSV or 复制）

   ![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/harbor/create-user-1.png)

   ![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/harbor/create-user-2.png)

3. 配置新用户的权限

   给当前用户添加`管理云解析(DNS)的权限`即可，因为需要做DNS验证。

   ![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/harbor/add-dns-ram.png)

**配置Access Key**

```shell
export Ali_Key="sdfsdfsdfljlbjkljlkjsdfoiwje"
export Ali_Secret="jlsdflanljkljlfdsaklkjflsa"
```

**申请证书**

```shell
acme.sh --issue --dns dns_ali -d gsealy.cn -d *.gsealy.cn
```

申请成功后，CA证书、csr、key、证书和证书链都会在指定目录下。比如我的就在`/root/.acme.sh/gsealy.cn`，申请的同时会自动配置定时任务，到既定时间会更新证书。

**复制证书**

我的机子不支持验证Let's Encrypt的证书，所以需要导入CA证书到Docker目录。先通过`install-cert`复制证书到一个常用目录

注：-d 就是申请的域名

```shell
acme.sh --install-cert -d gsealy.cn --install-cert --cert-file /opt/ssl/gsealy.cn/gsealy.cn.cer --ca-file /opt/ssl/gsealy.cn/ca.cer --key-file /opt/ssl/gsealy.cn/gsealy.cn.key --fullchain-file /opt/ssl/gsealy.cn/fullchain.cer
```

这样就把所有的证书都安装在指定目录了

## 部署Harbor

> 需要先安装Docker

### 首次安装

直接在Github下载最新的离线安装包，[地址](<https://github.com/goharbor/harbor/releases>)

**修改harbor.yml**

主要是修改hostname和https

```yaml
# 二级域名，DNS配置A类型解析为内网IP
hostname: harbor.gsealy.cn
https:
  # https port for harbor, default is 443
  port: 443
  # The path of cert and key files for nginx
  certificate: /opt/ssl/gsealy.cn/gsealy.cn.cer
  private_key: /opt/ssl/gsealy.cn/gsealy.cn.key
```

执行`./prepare`，然后执行`install.sh`即可

浏览器访问`harbor.gsealy.cn`，可以看到已经启用https，查看证书也是有效的，证书链也可以显示。

***** 但是在Docker中，需要配置CA证书，否则验证不通过

```shell
[root@localhost /]# docker login harbor.gsealy.cn
Username: xxxxxxx
Password: 
Error response from daemon: Get https://harbor.gsealy.cn/v2/: x509: certificate signed by unknown authority
```

此时需要复制CA证书到Docker目录，此种方法不需要重启Docker daemon

```shell
[root@localhost /]# cp /opt/ssl/gsealy.cn/ca.cer /etc/docker/certs.d/harbor.gsealy.cn
[root@localhost /]# cp /opt/ssl/gsealy.cn/gsealy.cn.cer /etc/docker/certs.d/harbor.gsealy.cn/
[root@localhost /]# cp /opt/ssl/gsealy.cn/gsealy.cn.key /etc/docker/certs.d/harbor.gsealy.cn/
```



### HTTP升级

**关闭所有运行harbor镜像**

ps. 不会删除数据，只是清理镜像

进入到harbor的工作目录（包含docker-compose.yml文件的目录），停止所有harbor运行镜像

```shell
docker-compose down -v
```

可选：**修改daemon.json**

若前期使用HTTP，在json中添加了`insecure-registries`配置，需要删除并重启docker服务

```
例：
"insecure-registries":["10.20.90.97:6145"]
```

按照**首次安装**进行部署

结束！🔚

------

