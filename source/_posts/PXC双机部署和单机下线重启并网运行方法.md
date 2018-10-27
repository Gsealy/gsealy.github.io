---
title: PXC双机部署和单机下线重启并网运行方法
tags:
  - PXC
  - MySQL
abbrlink: d3399d08
date: 2018-10-15 11:59:49
---

> 因为PhxSQL使用的时候，会出一些问题。所有机器都停掉再启动的时候，我就没有一步搞好过。文档也就是官方WIki里面的<成员管理>可以参考一下，基本就是重新部署了。

所以，又捡起了以前不知道什么原因放弃的PXC。部署起来也比较简单，一些不支持的东西也不太用的到。

> Note：
>
> 默认会用到下面的端口，生产环境记得配置相应的防火墙规则
>
> - 3306	**数据库对外服务的端口号**
> - 4444       **请求SST SST: 指数据一个镜象传输 xtrabackup , rsync ,mysqldump**
> - 4567       **组成员之间进行沟通的一个端口号**
> - 4568       **传输IST用的。相对于SST来说的一个增量**

```javascript
一些名词介绍：
WS：write set 写数据集
IST: Incremental State Transfer 增量同步
SST：State Snapshot Transfer 全量同步
```

**其他原理介绍可看：**

[MySQL高可用方案－PXC环境部署记录](https://cloud.tencent.com/developer/article/1026107)

------

### 一、安装

测试环境下，首先还是先把selinux和防火墙关掉，卸载系统中自带的MariaDB。

最方便的安装方式当然是`yum`了，自动安装依赖。先安装库：

```shell
sudo yum install http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm
```

再安装PXC就行了

```shell
sudo yum install Percona-XtraDB-Cluster-57 -y
```

启动MySQL

```shell
sudo service mysql start
```

因为5.7首次启动的密码叒改了，从`error.log`中抓初始密码即可

```shell
sudo grep 'temporary password' /var/log/mysqld.log
```

以`root`用户登录

```shell
mysql -u root -p
```

修改`root`账户密码

```shell
ALTER USER 'root'@'localhost' IDENTIFIED BY 'rootPass';
```

关掉MySQL服务，长时间无响应的话，kill掉相关pid就行了(ps. 其实service stop 也是直接 kill)。

```shell
sudo service mysql stop
```

修改`my.cnf`文件

```properties
#
# The Percona XtraDB Cluster 5.7 configuration file.
#
#
# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
#   Please make any edits and changes to the appropriate sectional files
#   included below.
#
!includedir /etc/my.cnf.d/
!includedir /etc/percona-xtradb-cluster.conf.d/
[mysqld]
datadir=/var/lib/mysql
user=mysql
  
# Path to Galera library
wsrep_provider=/usr/lib64/libgalera_smm.so
  
# Cluster connection URL contains the IPs of node#1, node#2 and node#3
wsrep_cluster_address=gcomm://192.168.85.147,192.168.85.148
  
# In order for Galera to work correctly binlog format should be ROW
binlog_format=ROW
  
# MyISAM storage engine has only experimental support
default_storage_engine=InnoDB
  
# This changes how InnoDB autoincrement locks are managed and is a requirement for Galera
innodb_autoinc_lock_mode=2
  
# Node #1 address
wsrep_node_address=192.168.85.147

# node name
wsrep_node_name=pxc1

# SST method
wsrep_sst_method=xtrabackup-v2
  
# Cluster name
wsrep_cluster_name=my_centos_cluster
  
# Authentication for SST method
wsrep_sst_auth="sstuser:s3cret"
```

不同节点主要修改`wsrep_node_address`和`wsrep_node_name`即可。

### 二、启动

我这里为了懒省事，直接完全克隆这个配置好的虚拟机。当做第二台PXC节点。

就修改里面的`my.cnf`即可。

**1、在`node#1`上使用命令启动PXC**

```shell
systemctl start mysql@bootstrap.service
```

进入`mysql`，添加同步所使用的账户密码，刷新权限。

```shell
mysql@pxc1> CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 's3cret';
mysql@pxc1> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO
  'sstuser'@'localhost';
mysql@pxc1> FLUSH PRIVILEGES;
```

<font color=red>**其中的账户密码，要和`my.cnf`中`wsrep_sst_auth`项所填的一样**</font>

**在`node#2`上**

2、直接启动mysql服务，<font color=red>**不可使用`mysql@bootstrap.service`启动**</font>

```shell
service mysql start
```

进入MySQL查看相应的参数 (仅展示需要查看的项)

```shell
mysql@pxc2> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
| ...                        | ...                                  |
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
| ...                        | ...                                  |
| wsrep_cluster_size         | 2                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| ...                        | ...                                  |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec)
```

看以上几项是否相同。集群大小是`2`，状态是`Primary`，本地是`Synced`状态。

这样两个node的PXC集群就搭建好了。

------

### 三、下线重启

当两台机子中有一台以及被迫下线的时候，可以简便的把修复好的节点并网继续运行。也不用使用`mysqldump`导入缺失的数据，直接启动服务就可以。

```shell
service mysql start
```

直接当子节点重启服气，因为在内部已经把剩余的一个node当做`mysql@bootstrap.service`服务在运行了。

查看两个node的状态，同时查看数据同步情况，在数据量不大的情况下，应该能够自动增量同步，依托于`percona`自家的`xtrabackup`。
```shell
mysql@pxc1> show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
| ...                        | ...                                  |
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
| ...                        | ...                                  |
| wsrep_cluster_size         | 2                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| ...                        | ...                                  |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec)
```

结束。🔚

------

