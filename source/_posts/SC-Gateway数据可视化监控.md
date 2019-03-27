---
title: SC-Gateway数据可视化监控
date: 2019-03-27 14:33:31
tags:
- Spring Boot 2
- Grafana
- Prometheus

---

### 前言

想把SC-Gateway的Metrics监控用起来，做到可视化监控。辅助日志监控。但是Spring官方的文档不适合刚上手Grafana的人。自己鼓捣了半天才知道咋整，记录一下。

> 当前版本：
>
> Spring Boot：2.1.3.RELEASE

### 下载

先要下载Grafana和Prometheus，直接去官网下最新版就行了。docker的话也直接pull least版本就行。

网址：[Grafana](https://grafana.com/) [Prometheus](https://prometheus.io/)

Docker: 

```bash
> docker pull grafana/grafana
> docker pull prom/prometheus
```

### 启动

Grafana启动bin目录下的`grafana-server`，在conf中可以自定义数据库、应用端口等内容，可以自己去看看。

Prometheus直接在根目录启动`prometheus`即可，也是可以通过`prometheus.yml`修改配置

### 视图

Grafana默认访问`3000`端口，Prometheus默认访问`9090`端口就能看到网页。

tip. 但Grafana第一次启动需要建库建表可能需要点时间，如果刷新不管用的话，可以在启动的cli中回车一下。

### 配置

1、配置Spring项目

先在pom中添加dependency：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
	<groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.1.3</version>
</dependency>
```

暴露Endpoint：

ps. 按需暴露端点，在这里图省事就全暴露了

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

2、配置Prometheus

修改prometheus.yml文件，就在最后加一个新的job即可，主要关注job_name、metrics_path和targets，分别是任务名，metrics路径和需要监控的IP

```yaml
global:
  - job_name: 'prometheus'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['127.0.0.1:12305']
```

重启Prometheus，访问：[127.0.0.1:9090/targets](http://127.0.0.1:9090/targets) 就可以看到端点和状态

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/promethea_target.png)

3、配置Grafana

先访问[http://127.0.0.1:3000](http://127.0.0.1:3000/)，使用默认的账号admin/admin，登录系统，可以改密也可以先跳过，然后就会定位到如下页面，选择Prometheus数据源

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/grafana_configuration.png)

配置数据源，Name要和prometheus.yml中的job_name相同，url就是prometheus的地址，其他不用配置，点击`Save&Test`，成功的话就可以进行下一步了。

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/grafana_datasource.png)

这里直接使用SC-Gateway提供的json模板，所以不按照步骤进行下一步，直接跳过配置panel。点选import

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/grafana_import_1.png)

模板地址: [link](https://raw.githubusercontent.com/spring-cloud/spring-cloud-gateway/master/docs/src/main/asciidoc/gateway-grafana-dashboard.json)

点选配置好的prometheus数据源，选择合适的文件夹导入就可以了

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/grafana_import_3.png)

当你的数据源没有问题的时候，dashboard所有信息都会展示出来。如果有什么东西有问题的话，比如数据源配置有错，就需要重新检查数据源配置。

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/grafana_dashboard.png)

结束！🔚

------

