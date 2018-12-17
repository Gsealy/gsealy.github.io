---
title: 更换Spring Boot 2.0的连接池
date: 2018-12-17 11:15:46
tags:
- Pool
- Spring Boot
---

> **前言**
>
> 前期做gateway demo的时候一直用的都是阿里的`Druid`，自带监控，但是暂时监控用不上，就想试试`HikariCP`，所以要将`Druid`转移至`HikariCP`连接池
>
> 当前Spring Boot版本为：**2.1.0.RELEASE**

#### `HikariCP`简介

HikariCP专注于连接池，没有加Druid中的监控功能，轻量级，代码一共只有130kb左右

**Github地址:**  [https://github.com/brettwooldridge/HikariCP](https://github.com/brettwooldridge/HikariCP)

Spring Boot 2.0将`HikariCP`作为默认的连接池，给出了如下解释：

> Production database connections can also be auto-configured by using a pooling`DataSource`. Spring Boot uses the following algorithm for choosing a specific implementation:
>
> 1. We prefer [HikariCP](https://github.com/brettwooldridge/HikariCP) for its performance and concurrency. If HikariCP is available, we always choose it.
> 2. Otherwise, if the Tomcat pooling `DataSource` is available, we use it.
> 3. If neither HikariCP nor the Tomcat pooling datasource are available and if [Commons DBCP2](https://commons.apache.org/proper/commons-dbcp/) is available, we use it.

#### 引入依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>${mysql.version}</version>
      <scope>runtime</scope>
</dependency>
```

**Tips：**没有在`spring-boot-starter-jdbc`的`pom`文件里发现`tomcat-jdbc`，可能默认的就已经是`HikariCP`了吧，所以也不用排除`tomcat-jdbc`

#### yaml配置

```yaml
spring:
  datasource:
  driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/database?useUnicode=true&characterEncoding=utf-8&useSSL=false
    username: root
    password: root
    hikari:
      auto-commit: true
      maximum-pool-size: 60
      minimum-idle: 10
      connection-test-query: SELECT 1
      connection-timeout: 30000
      idle-timeout: 600000
      maxLifetime: 1800000
      read-only: false
    
```

这样直接启动就可以了，会看到`HikariCP`启动成功

```bash
[2018-12-17 11:34:48] [RMI TCP Connection(6)-127.0.0.1] INFO  [com.zaxxer.hikari.HikariDataSource] 110 - HikariPool-1 - Starting...
[2018-12-17 11:34:48] [RMI TCP Connection(6)-127.0.0.1] DEBUG [com.zaxxer.hikari.pool.HikariPool] 545 - HikariPool-1 - Added connection com.mysql.jdbc.JDBC4Connection@3961e5e4
[2018-12-17 11:34:48] [RMI TCP Connection(6)-127.0.0.1] INFO  [com.zaxxer.hikari.HikariDataSource] 123 - HikariPool-1 - Start completed.
```

##### 配置上的坑

起初配置的时候，是如下这样的

```yaml
spring:
  datasource:
    hikari:
      jdbc-url: jdbc:mysql://127.0.0.1:3306/database?useUnicode=true&characterEncoding=utf-8&useSSL=false # <1>
      username: root                                 # <2>
      password: root                                 # <3>
      driver-class-name: com.mysql.jdbc.Driver       # <4>
      auto-commit: true
      maximum-pool-size: 60
      minimum-idle: 10
      connection-test-query: SELECT 1
      connection-timeout: 30000
      idle-timeout: 600000
      maxLifetime: 1800000
      read-only: false
```

当把上面这四项放在`spring.datasource.hikari`下面的时候，启动会报如下错误：

```java
org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Failed to determine a suitable driver class
...
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine a suitable driver class


Action:

Consider the following:
	If you want an embedded database (H2, HSQL or Derby), please put it on the classpath.
	If you have database settings to be loaded from a particular profile you may need to activate it (no profiles are currently active).
```

“无法确定合适的驱动类”，配置`DataSource`下的`'url'`参数失败，像阿里自己封装的`druid-starter`，所有都配置`spring.datasource.druid`在即可，所以上面的解决办法就是把打了注释这四项提出到上一级就可以了，顺道把`jdbc-url`改为`url`就可以用了。

还有一个小问题就是`IDEA`针对`spring.datasource.driver-class-name`这一项配置居然不识别，虽然报错，但是不影响使用

> IDEA当前版本：IntelliJ IDEA 2018.3.1 (Ultimate Edition)

![](https://ws1.sinaimg.cn/large/7074e5d2ly1fy9pd9022jj20nu02eglo.jpg)

