---
title: Spring cloud Gateway两种Route动态配置方式
tags:
  - Spring Cloud
  - Gateway
  - Dynamic Route
abbrlink: d49605fb
date: 2018-10-18 10:05:15
---

> **前言**
> 最近在用Spring Cloud Gateway（sc-gateway）的时候，总是被他的Route编辑方式搞的很难受，只能写死。网上找了找，有两个实现的方式，还有一个写的不是很全。所以自己整理了一下。

通过这篇 [Spring Cloud Gateway运行时动态配置网关](https://my.oschina.net/tongyufu/blog/1844573)，了解了基本动态配置的方式。万分感谢。但是写的不是很详细。也看到评论有不能复现的。

### sc-gateway 支持动态配置么？

查看上面的blog，可以知道是支持的，也支持RESTful方式，内部写好了相应的类，就是现今文档不是很详细。源码的javadoc也写的很模糊。`GatewayControllerEndpoint`类中，只是很简单的写了个注释。

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/gateway-original-method.jpg)

因为这种方式依赖于健康检查，先在`pom.xml`里面添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

再在application.yml中添加，以打开配置访问

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

默认打开了`consul`的`json`配置

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/gateway-actuator.jpg)

### 重新实现动态配置

以后肯定是要关闭健康检查中的配置节点的，所以要重新覆写一个api了。

实现`ApplicationEventPublisherAware`，创建一个刷新`Route`的发布事件

```java
import org.springframework.cloud.gateway.event.RefreshRoutesEvent;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.ApplicationEventPublisherAware;
import org.springframework.stereotype.Component;

/**
 * 通过实现ApplicationEventPublisherAware来发布路由刷新事件
 *
 * @author Gsealy
 * @since 2018-10-17 14:07:03
 */
@Component
public class GatewayRoutesRefresher implements ApplicationEventPublisherAware {

  private ApplicationEventPublisher publisher;

  @Override
  public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
    this.publisher = applicationEventPublisher;
  }

  public void refreshRoutes() {
    publisher.publishEvent(new RefreshRoutesEvent(this));
  }
}
```

再创建一个用来配置路由信息的`RouteLocator`，其中有两种配置方式，第一种是通过`RouteLocator.Builder`的Build+lambda表达式，但是只能给出默认的配置，针对多种的filter不好给出参数。

```java
public RefreshRouteLocator addRoute(@NotNull final String id, @NotNull final String path, @NotNull final URI uri)
```

另一种是通过`RouteDefinition`，通过自建对象（手动滑稽）的方式，添加各种的配置。但是其中的`Key`是**大小写敏感**的

```java
public RefreshRouteLocator addRoute(@NotNull RouteDefinition definition)
```

```java
import java.net.URI;
import java.net.URISyntaxException;
import javax.validation.constraints.NotNull;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.cloud.gateway.route.RouteDefinitionWriter;
import org.springframework.cloud.gateway.route.RouteLocator;
import org.springframework.cloud.gateway.route.builder.RouteLocatorBuilder;
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

/**
 * 动态加载路由配置
 *
 * @author Gsealy
 * @since 2018-10-17 14:07:24
 */
@Component
public class RefreshRouteLocator implements RouteLocator {

  private RouteLocatorBuilder builder;
  private RouteLocatorBuilder.Builder routesBuilder;
  private Flux<Route> route;

  @Autowired
  GatewayRoutesRefresher gatewayRoutesRefresher;

  @Autowired
  private RouteDefinitionWriter routeDefinitionWriter;

  public RefreshRouteLocator(RouteLocatorBuilder builder) {
    this.builder = builder;
    clearRoutes();
  }

  public void clearRoutes() {
    routesBuilder = builder.routes();
  }

  /**
   * 使用RouteLocatorBuilder.Builder创建新的路由规则（ps.仅支持添加最基础的转发规则）
   *
   * @param id 路由id
   * @param path 路由path
   * @param uri 指向地址
   * @return RefreshRouteLocator
   */
  @NotNull
  public RefreshRouteLocator addRoute(@NotNull final String id, @NotNull final String path,
      @NotNull final URI uri) throws URISyntaxException {
    if (StringUtils.isEmpty(uri.getScheme())) {
      throw new URISyntaxException("Missing scheme in URI: {}", uri.toString());
    }

    routesBuilder.route(id, fn -> fn
        .path(path + "/**")
        .uri(uri)
    );

    return this;
  }

  /**
   * 使用RouteDefinition添加路由节点，可自己配置相关属性
   *
   * @param definition 属性定义
   * @return RefreshRouteLocator
   */
  @NotNull
  public RefreshRouteLocator addRoute(@NotNull RouteDefinition definition) {
    routeDefinitionWriter.save(Mono.just(definition)).subscribe();
    return this;
  }

  /**
   * 配置完成后，调用本方法构建路由和刷新路由表
   */
  public void buildRoutes() {
    if (routesBuilder != null) {
      this.route = routesBuilder.build().getRoutes();
    }
    gatewayRoutesRefresher.refreshRoutes();
  }

  /**
   * @return 路由信息
   */
  @Override
  public Flux<Route> getRoutes() {
    return route;
  }
}
```

这样，直接在Controller里面添加路由信息即可，演示一下第二种方式

```java
import java.net.URI;
import java.util.Arrays;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.gateway.handler.predicate.PredicateDefinition;
import org.springframework.cloud.gateway.route.RouteDefinition;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.util.UriComponentsBuilder;

@Slf4j
@Controller
public class RouteController {

  @Autowired
  private RefreshRouteLocator refreshableRoutesLocator;

  private Map<String /*Tag*/, String /*Path*/> routes = new ConcurrentHashMap<>();

  @ResponseBody
  @PostMapping("/testbaidu")
  public String baidu() {
    RouteDefinition definition = new RouteDefinition();        // <1>
    PredicateDefinition predicate = new PredicateDefinition(); // <2>
    Map<String, String> predicateParams = new ConcurrentHashMap<>(8);

    definition.setId("baiduRoute");
    predicate.setName("Path");
    predicateParams.put("pattern", "/baidu");
    predicateParams.put("pathPattern", "/baidu");
    predicate.setArgs(predicateParams);
    definition.setPredicates(Arrays.asList(predicate));
    URI uri = UriComponentsBuilder.fromHttpUrl("https://www.baidu.com").build().toUri();
    definition.setUri(uri);
    refreshableRoutesLocator.addRoute(definition);            // <3>
    log.info("添加的代理路径: tag {} -> {}", "/baidu", uri);
    refreshableRoutesLocator.buildRoutes();                   // <4>
    log.info("创建完成");
    return "success";
  }

}
```

代码中的<1>和<2>，就是`RouteDefinition`，`PredicateDefinition`两种定义对象

```bash
http post http://127.0.0.1:12305/testbaidu
```

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/gateway-httpie-1.jpg)

可以拉取一下配置文件看看

```bash
http get http://127.0.0.1:12305/actuator/gateway/routes/
```

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/gateway-httpie-2.jpg)

这样就初步完成配置

### 怎么启动后没有静态配置的Route信息了？

在抓取配置信息的时候，出现了500错误，出现了`The mapper returned a null Publisher`的error信息。明明在`application.yml`或者代码中配置了route信息。只有在添加新的路由信息后，才能刷新出路由表。

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/gateway-httpie-3.jpg)

这样需要在Spring boot启动的时候，自动去刷新一下路由表，为了方便后期在添加启动执行项，创建一个`startup`类，在里面调用一下`buildRoutes`

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

/**
 * 启动后自刷新路由
 * @author Gsealy
 * @since 2018-10-18 10:01:50
 */
@Component
public class GatewayStartup {

  @Autowired
  private RefreshRouteLocator refreshRouteLocator;

  @Bean
  public GatewayStartup createApplicationStartup() {
    return new GatewayStartup();
  }

  public void start() {
    refreshRouteLocator.buildRoutes();
  }
}
```

在启动类中添加`startup`方法，配置相应的注解，从而Spring Boot启动后执行`startup`方法。启动后就会自动刷新路由信息。

```java
/**
 * @author Gsealy
 */
@EnableDiscoveryClient
@SpringBootApplication
public class GatewayApplication {

  @Autowired
  private GatewayStartup gatewayStartup;

  @EventListener(ApplicationReadyEvent.class)
  public void startup() {
    gatewayStartup.start();
  }

  public static void main(String[] args) {
    SpringApplication.run(GatewayApplication.class, args);
  }
}
```

至此，基本配置完成，可以再在上面搭建前端展示页面。

（Ps. 要是配置的静态资源啥的访问不了，也是路由表的问题，刷新后就好了）

> #### 引用：
>
> [Spring Cloud Gateway运行时动态配置网关](https://my.oschina.net/tongyufu/blog/1844573)
>
> [Persisting Spring Cloud Gateway Routes in Database](https://stackoverflow.com/a/51499046/9137803) 🔚

------

