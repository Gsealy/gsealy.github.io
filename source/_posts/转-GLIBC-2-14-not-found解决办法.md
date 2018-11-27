---
title: '[转]''GLIBC_2.14'' not found解决办法'
tags:
  - Kong
  - CentOS
  - 转载
abbrlink: 6c4e7b5a
date: 2018-09-26 09:02:41
---

最近在CentOS6上部署Kong 0.14时，遇到了这个问题。所以找到解决办法以后记录一下。

### 一、下载、编译、安装

下载[glibc-2.14.tar.gz](http://ftp.gnu.org/gnu/glibc/glibc-2.14.tar.gz)或者[百度云](https://pan.baidu.com/s/1xYOrZtl46t_48flWUBTozA 提取码: ksv6)

编译并安装

```bash
[root@localhost ~]# tar zxvf glibc-2.14.tar.gz -C /home/software/
[root@localhost ~]# cd /home/software/glibc-2.14
[root@localhost glibc-2.14]# mkdir /opt/build
[root@localhost glibc-2.14]# cd build
[root@localhost build]# ../configure --prefix=/opt/glibc-2.14
[root@localhost build]# make -j4
[root@localhost build]# make install
```

==编译安装时间稍长，需要耐心等待==

### 二、中间遇到的坑

1、在make过程中出现如下错误：

```bash
/usr/bin/install: 'include/limits.h' and '/opt/glibc-2.14/include/limits.h' are the same file
```

原因就是楼主解压的glic-2.14.tar.gz源码和编译时定义的目录../configure --prefix=/home/software/glibc-2.14放到了一起。

所以解决方法就是：只要将编译定义目录和源码目录区分开就ok了。

2、最后就是设置环境变量，因为glibc库使用广泛，为了避免污染当前系统环境，在使用时候定义一下环境变量。

```bash
[root@localhost ~]# export LD_LIBRARY_PATH=/opt/glibc-2.14/lib:$LD_LIBRARY_PATH
```

将库的位置临时定位在/opt/glibc-2.14/lib位置。

此时再执行相关程序即可顺利运行。

> 转载自：https://blog.csdn.net/clirus/article/details/62425498

结束！🔚

------

