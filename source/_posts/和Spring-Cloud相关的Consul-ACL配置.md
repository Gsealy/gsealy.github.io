---
title: 和Spring Cloud相关的Consul ACL配置
date: 2019-08-08 10:51:32
tags:
 - ACL
 - Spring Cloud
---

## 前提

基于Spring Cloud构建的应用如果需要跨Agent调用，还需要配置不同的规则，在这里记录下踩坑。

## 构建项目

创建一个应用，起名`cloud-service-two`， 只需要注册`cloud-service-two`即可，相关配置

```properties
spring.application.name=cloud-service-two
spring.cloud.consul.host=10.20.88.35
spring.cloud.consul.port=8500
spring.cloud.consul.discovery.instance-id=${spring.application.name}-${spring.cloud.consul.discovery.ip-address}-${server.port}
spring.cloud.consul.discovery.prefer-ip-address=true
spring.cloud.consul.discovery.acl-token=e0db129f-8123-2b0d-f265-c4a6f19fb850
spring.cloud.consul.discovery.ip-address=10.20.61.24
```

## ACL配置

1. service two 配置

因为service需要注册，所以需要Agent的写权限。创建一个`services-write`的策略

```
service "cloud-service-two" { policy = "write" }
```

然后分配token，填写在配置文件里面，这样应用就可以去向Consul注册了

2. service one配置

**注意**：这里就是踩坑点，本以为给配置对应node和service的`read`策略就可以，远没有想象的那么简单。

先看下`ConsulDiscoveryClient`类，当前类会实现`DiscoveryClient`接口，当有操作需要做服务发现调用时，直接使用这个接口即可。（这里可能理解有误，欢迎指正！），调用关系如下

```java
@Override
getInstances() -> addInstancesToList()
```

通过传入`应用名（serviceName）`来获取对应服务的IP端口集合，具体代码如下所示:

```java
private void addInstancesToList(List<ServiceInstance> instances, String serviceId,
			QueryParams queryParams) {
	String aclToken = this.properties.getAclToken();
	Response<List<HealthService>> services;
	if (StringUtils.hasText(aclToken)) {
		services = this.client.getHealthServices(serviceId,                 // 1
							this.properties.getDefaultQueryTag(),
							this.properties.isQueryPassing(), queryParams, aclToken);
	} else {
		services = this.client.getHealthServices(serviceId,
							this.properties.getDefaultQueryTag(),
							this.properties.isQueryPassing(), queryParams);
	}
	for (HealthService service : services.getValue()) {
		String host = findHost(service);
		Map<String, String> metadata = getMetadata(service);
		Boolean secure = false;
		if (metadata.containsKey("secure")) {
			secure = Boolean.parseBoolean(metadata.get("secure"));
		}
		instances.add(new DefaultServiceInstance(service.getService().getId(),
							serviceId, host, service.getService().getPort(), secure, metadata));
	}
}
```

`1`处指出，Spring Cloud是通过健康检查服务（地址：`/v1/health/service/:serviceName`）去获取当前服务名下注册的所有服务，具体请看：[List Nodes for Service接口详解](https://www.consul.io/api/health.html#list-nodes-for-service)

从官方文档可以看到，这里ACL需要**节点可读，服务可读**，所以我们的策略也应该这么去写。（之前就是只给了单独服务的可读权限一直就获取不到），策略如下：

```
# 安全性低, 所有节点和服务名都可以暴露，基本等同于Mater Token
node_prefix "" { policy = "read" }
service_prefix "" { policy = "read" }

# 仅开放相关node和service的可读权限
node "client-node-1" { policy = "read" }
service "cloud-service-two" { policy = "read" }
```

使用这个策略生成一个Token，测试一下：

```shell
> http :8500/v1/health/service/cloud-service-two X-Consul-Token:55aacb1b-8d85-0289-8c13-d85c cf55fd2e
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Length: 741
Content-Type: application/json
Date: Thu, 08 Aug 2019 06:18:34 GMT
Vary: Accept-Encoding
X-Consul-Effective-Consistency: leader
X-Consul-Index: 27007
X-Consul-Knownleader: true
X-Consul-Lastcontact: 0

[
    {
        "Checks": [
            {
                "....": "...",
                      .
                      .
                      .
                "....": "...."
            },
            {
                ....
            }
        ],
        "Node": {
            ....
        },
        "Service": {
            ....
        }
    }
]
```

顺利通过！

## 相关官方文档

[1] ACL Rules：[https://www.consul.io/docs/acl/acl-rules.html](https://www.consul.io/docs/acl/acl-rules.html)

[2] ACL System: [https://www.consul.io/docs/acl/acl-system.html](https://www.consul.io/docs/acl/acl-system.html)

[3] Configuration: [https://www.consul.io/docs/agent/options.html](https://www.consul.io/docs/agent/options.html)

[4] Authentication: [https://www.consul.io/api/index.html#authentication](https://www.consul.io/api/index.html#authentication)

结束！🔚

------

