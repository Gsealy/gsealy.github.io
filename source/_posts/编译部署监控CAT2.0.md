---
title: 编译部署监控CAT2.0
tags:
  - CAT
  - 监控
abbrlink: d4222fd6
date: 2018-10-09 17:27:39
---

最近还是在研究微服务，看到了链路追踪这部分。发现点评开源的CAT比较完成，可以直接集成使用。所以就先盯上了CAT。

### 一、编译

因为我的测试用服务器是不联网的，所以先在本地编译好war包。

1、git克隆cat项目到本地

```bash
git clone https://github.com/dianping/cat.git
```

2、直接编译

```bash
mvn clean install
```

这样，在`cat-home/target/`下就有一个`cat-2.0.0.war`(or `cat-alpha-2.0.0.war`)

### 二、部署

从官网下载[tomcat](http://mirrors.hust.edu.cn/apache/tomcat/tomcat-8/v8.5.34/bin/apache-tomcat-8.5.34.tar.gz)和[jdk1.8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，删除自带的其他版本java，保证war包能够运行就可以了。

解压缩`tomcat`，路径自选，进入`bin`目录，按照官网给出的修改`catalina.sh`文件(`ps.`刚发现他们在升级3.0.0版，readme给改了0.0，那下个2.0.0的release版吧)

在`# OS specific support.  $var _must_ be set to either true or false.`后一行添加`CATALINA_OPTS`参数![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/cat/cat-catalina.jpg)



***ps.* 建议cat的使用堆大小至少10G以上，开发环境启动2G堆启动即可**

```shell
CATALINA_OPTS="$CATALINA_OPTS -server -Djava.awt.headless=true -Xms25G -Xmx25G -XX:PermSize=256m -XX:MaxPermSize=256m -XX:NewSize=10144m -XX:MaxNewSize=10144m -XX:SurvivorRatio=10 -XX:+UseParNewGC -XX:ParallelGCThreads=4 -XX:MaxTenuringThreshold=13 -XX:+UseConcMarkSweepGC -XX:+DisableExplicitGC -XX:+UseCMSInitiatingOccupancyOnly -XX:+ScavengeBeforeFullGC -XX:+UseCMSCompactAtFullCollection -XX:+CMSParallelRemarkEnabled -XX:CMSFullGCsBeforeCompaction=9 -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSClassUnloadingEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:-ReduceInitialCardMarks -XX:+CMSPermGenSweepingEnabled -XX:CMSInitiatingPermOccupancyFraction=70 -XX:+ExplicitGCInvokesConcurrent -Djava.nio.channels.spi.SelectorProvider=sun.nio.ch.EPollSelectorProvider -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.util.logging.config.file="%CATALINA_HOME%\conf\logging.properties" -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCApplicationConcurrentTime -XX:+PrintHeapAtGC -Xloggc:/data/applogs/heap_trace.txt -XX:-HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/applogs/HeapDumpOnOutOfMemoryError -Djava.util.Arrays.useLegacyMergeSort=true"
```

修改`server.xml`文件，添加`UTF-8`

```xml
<Connector port="8080" protocol="HTTP/1.1"
           URIEncoding="utf-8"    connectionTimeout="20000"
               redirectPort="8443" />  增加  URIEncoding="utf-8"
```

把在本地编译好的`cat-*-2.0.0.war`改名为`cat.war`放进webapps目录下

##### 1、在根目录下创建具有读写权限的目录

```shell
mkdir /data
chmod 777 /data/ -R
```

##### 2、编写配置文件（单机）

配置文件创建在`/data/appdatas/cat/`内

**client.xml** (配置受监控的客户端)

配置监控的服务器本机和客户端IP，监控客户端`2280`端口

```xml
<?xml version="1.0" encoding="utf-8"?>
<config mode="client">
	<servers>
        <!-- server本机 IP-->
		<server ip="10.20.88.222" port="2280" http-port="8080"/>
        <!-- 远程client IP-->
		<server ip="10.20.61.19" port="2280" />
	</servers>
</config>
```

**server.xml**（配置服务端）

因为是单机配置，所以`job-machine`和`alert-machine`都在本机开启。关闭hdfs存储。其余参数参考官方README.md

```xml
<?xml version="1.0" encoding="utf-8"?>
<config local-mode="false" hdfs-machine="false" job-machine="true" alert-machine="true">
        <storage  local-base-dir="/data/appdatas/cat/bucket/" max-hdfs-storage-time="15" local-report-storage-time="7" local-logivew-storage-time="7">
                <!--<hdfs id="logview" max-size="128M" server-uri="hdfs:///user/cat" base-dir="logview"/>
                <hdfs id="dump" max-size="128M" server-uri="hdfs://10.1.77.86/user/cat" base-dir="dump"/>
                <hdfs id="remote" max-size="128M" server-uri="hdfs://10.1.77.86/user/cat" base-dir="remote"/>-->
        </storage>
        <console default-domain="Cat" show-cat-domain="true">
                <remote-servers>10.20.88.222:8080</remote-servers>
        </console>
</config>

```

**datasources.xml**（配置数据源）

修改里面的URI和账户密码即可

```xml
<data-sources>
	<data-source id="cat">
		<maximum-pool-size>3</maximum-pool-size>
		<connection-timeout>1s</connection-timeout>
		<idle-timeout>10m</idle-timeout>
		<statement-cache-size>1000</statement-cache-size>
		<properties>
			<driver>com.mysql.jdbc.Driver</driver>
			<url>
				<![CDATA[jdbc:mysql://127.0.0.1:3306/cat]]>
			</url>
			<user>root</user>
			<password>root</password>
			<connectionProperties>
				<![CDATA[useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&socketTimeout=120000]]>
			</connectionProperties>
		</properties>
	</data-source>
	<data-source id="app">
		<maximum-pool-size>3</maximum-pool-size>
		<connection-timeout>1s</connection-timeout>
		<idle-timeout>10m</idle-timeout>
		<statement-cache-size>1000</statement-cache-size>
		<properties>
			<driver>com.mysql.jdbc.Driver</driver>
			<url>
				<![CDATA[jdbc:mysql://10.20.88.222:3306/cat]]>
			</url>
			<user>root</user>
			<password>root</password>
			<connectionProperties>
				<![CDATA[useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&socketTimeout=120000]]>
			</connectionProperties>
		</properties>
	</data-source>
</data-sources>

```

##### 3、建库建表

手动安装`mysql`，MySQL的一个系统参数：max_allowed_packet，其默认值为1048576(1M)，修改为1000M，,修改`my.cnf`，添加如下一行。修改完需要重启MySQL

```bash
max_allowed_packet=1000M
```

然后，执行项目下`script/Cat.sql`即可

### 三、启动

定位到`tomcat/bin`目录下，执行

```bash
./catalina.sh run
```

打开：http://ip:8080/cat/s/config?op=routerConfigUpdate 

修改`default-server`为本机ip，重启tomcat即可

```xml
<?xml version="1.0" encoding="utf-8"?>
<router-config backup-server="10.20.88.222" backup-server-port="2280">
   <default-server id="10.20.88.222" weight="1.0" port="2280" enable="true"/>
</router-config>
```

至此，服务端配置完成。🔚

------

