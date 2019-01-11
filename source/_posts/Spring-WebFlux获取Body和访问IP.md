---
title: Spring WebFlux获取Body和访问IP
date: 2019-01-11 10:55:52
tags:
- Spring Boot
- WebFlux
- Reactor
---

> 当通过`subscribe`获取body体的时候，总是报null。
>
> Spring Boot版本：2.1.1.RELEASE

### 官方样例

`Spring`官方有一个样例：[spring-boot-sample-webflux](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-webflux)

包含一个启动类、2个Controller和一个Handler。

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/class.png)

`Controller`是两个基本的基于`@RestController`注解的Controller类，与以往`@Controller`注解的区别就是包含了`@RequestBody`注解。这次主要就是看看这个`Handler`类

```java
@Component
public class EchoHandler {

	public Mono<ServerResponse> echo(ServerRequest request) {
		return ServerResponse.ok().body(request.bodyToMono(String.class), String.class);
	}

}
```

```java
@Bean
public RouterFunction<ServerResponse> monoRouterFunction(EchoHandler echoHandler) {
	return route(POST("/echo"), echoHandler::echo);
}
```

通过`Bean`注册路由信息，这和以往的`@Controller`有所不同，但是和`Vert.x`很相似。在`Handler`中通过`ServerRequest`接口接收请求信息。

```java
// Vert.x 路由部署方式
...
router.get("/").handler(this::indexhandler);
router.route().handler(BodyHandler.create().setMergeFormAttributes(true));
router.route("/static/*").handler(StaticHandler.create("static"));
...
```

**运行样例：**

使用`Httpie`携带Body 信息请求`/echo`，返回请求的Body信息

```bash
E:\spring-boot-sample-webflux>http post :8080/echo foo=bar
HTTP/1.1 200 OK
Content-Length: 14
Content-Type: text/plain;charset=UTF-8

{
    "foo": "bar"
}

```

### 获取请求

上面的实例可以获取Body信息，但是当我们想`subscribe`获取这个body内的信息时，会得到一个`null`的对象。

错误代码：

```java
request.bodyToMono(String.class)
       .subscribe(body -> {
          System.out.println("body data->" + body);
        });
```

在[Spring Boot#15320](https://github.com/spring-projects/spring-boot/issues/15320#issuecomment-442574935)找到了答案

```
This code is calling subscribe on the request body and decouples it from the response rendering. Because you're decoupling the request handling from the bit that reads the request body, you're running into a race condition: by the time the response is handled, Spring WebFlux is cleaning the HTTP resources (request and response resources), which means that your other subscription might not have time to read the body.
```

主要意思就是：

```
调用subscribe()方法获取请求体会割裂响应。此时正处于竞争态，在处理响应时，Spring WebFlux正在清理HTTP资源（请求和响应资源），意味着你其他的订阅无法得到请求体。
```

所以想对Body做操作时，需要链式调用，最好不要使用订阅等方法，而且这里还处于竞争态，更不能使用。所以单独获取Body或者某一请求值的话，直接使用`flatMap()`方法

```java
public Mono<ServerResponse> test2(ServerRequest request) {
  return request.bodyToMono(String.class)
      .flatMap(s -> ServerResponse.ok().body(Mono.just(s), String.class));
}
```

这样就可以在`flatMap()`操作body了，同时在后面加上了`log()`。看下调用情况

```bash
2019-01-11 15:19:25.971  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.FlatMap.2                   : | onSubscribe([Fuseable] MonoFlatMap.FlatMapMain)
2019-01-11 15:19:25.971  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.FlatMap.2                   : | request(unbounded)
2019-01-11 15:19:25.975  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.OnErrorResume.1             : onSubscribe(FluxOnErrorResume.ResumeSubscriber)
2019-01-11 15:19:25.975  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.OnErrorResume.1             : request(unbounded)
2019-01-11 15:19:25.984  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.OnErrorResume.1             : onNext({"foo": "bar"})   
2019-01-11 15:19:25.990  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.FlatMap.2                   : | onNext(org.springframework.web.reactive.function.server.DefaultEntityResponseBuilder$DefaultEntityResponse@b65494c)
2019-01-11 15:19:26.017  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.FlatMap.2                   : | onComplete()
2019-01-11 15:19:26.017  INFO 19612 --- [ctor-http-nio-2] reactor.Mono.OnErrorResume.1             : onComplete()
```

可以在第五行看到`onNext({"foo": "bar"})`,显示出了请求体。同时可以看到，`reactor.Mono.OnErrorResume.1`这里是调用了`OnErrorResume`方法，我理解就是存在竞争，它是从错误中恢复出来的。

当获取多个参数时，需要使用`zip()`方法将几个参数zip在一起，例如获取请求IP和请求体

```java
public Mono<ServerResponse> test(ServerRequest request) {
  return Mono.zip(request.bodyToMono(String.class),
      Mono.just(request.remoteAddress()
          .map(InetSocketAddress::getHostString)
          .orElseThrow(RuntimeException::new)))
        .flatMap(tuple -> {
          String bodyData = tuple.getT1();
          String remoteIp = tuple.getT2();
          log.info("BodyData =>" + bodyData);
          log.info("RemoteIp =>" + remoteIp);
          return ServerResponse.ok().body(Mono.just(bodyData + "\n" + remoteIp), String.class);
        });
  }
```

请求`/test`端点

```bash
C:\spring-boot-sample-webflux>http post :8080/test foo=bar
HTTP/1.1 200 OK
Content-Length: 30
Content-Type: text/plain;charset=UTF-8

{"foo": "bar"}
0:0:0:0:0:0:0:1

```

### 总结

​	想要在Spring 5，Spring Boot 2上得到更好的体验。可以尝试转到流式编程上，包括使用`lambda`表达式等。可以增加代码的可读性和整洁性。

​	在这个例子里面，主要是获取多个请求参数，我自己也绕了好久，从以往的注解中跳出来。更多的可以看看`Pivotal`开源的`Reactor`或者`RxJava`等等。

结束！🔚

------

