---
title: HTTP慢速攻击下webflux化解方式
tags:
  - Webflux
  - Netty
abbrlink: 857074dc
date: 2021-06-09 14:00:04
---

# 前言

业务应用在做漏扫时发现一个中危漏洞需要处理。采用的是Spring Cloud Webflux，使用内嵌的 Netty 作为Web容器。因为是响应式编程，漏扫工具也没有提供对应的解决办法。

![漏扫结果](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/scan-result.png)

## 介绍

缓慢的HTTP拒绝服务攻击是一种专门针对于Web的应用层拒绝服务攻击，攻击者操纵网络上的肉鸡，对目标Web服务器进行海量HTTP请求攻击，直到服务器带宽被打满，造成了拒绝服务。

慢速HTTP拒绝服务攻击经过不断的演变和发展，主要有三种攻击类型，分别是Slow headers、Slow body、Slow read。以Slow headers为例，Web应用在处理HTTP请求之前都要先接收完所有的HTTP头部，因为HTTP头部中包含了一些Web应用可能用到的重要的信息。攻击者利用这点，发起一个HTTP请求，一直不停的发送HTTP头部，消耗服务器的连接和内存资源。抓包数据可见，攻击客户端与服务器建立TCP连接后，每10秒才向服务器发送一个HTTP头部，而Web服务器在没接收到2个连续的\r\n时，会认为客户端没有发送完头部，而持续的等等客户端发送数据。如果恶意攻击者客户端持续建立这样的连接，那么服务器上可用的连接将一点一点被占满，从而导致拒绝服务。这种攻击类型称为慢速HTTP拒绝服务攻击。

# 处理

Netty可以配置TCP的读写超时，但是测试后无效。需要添加IdleHandler控制HTTP的空闲状态。

```java
@Configuration
public class ServerConfig {

  @Bean
  public ReactiveWebServerFactory reactiveWebServerFactory(
      @Value("${server.netty.idle-timeout}") Duration idleTimeout) {

    final NettyReactiveWebServerFactory factory = new NettyReactiveWebServerFactory();
    factory.addServerCustomizers(server ->
        server.tcpConfiguration(tcp ->
            tcp.bootstrap(bootstrap -> bootstrap.childHandler(new ChannelInitializer<>() {
              @Override
              protected void initChannel(Channel channel) {
                channel.pipeline().addLast(
                    new IdleStateHandler(0, 0, idleTimeout.toNanos(), NANOSECONDS),
                    new ChannelDuplexHandler() {
                      @Override
                      public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
                        if (evt instanceof IdleStateEvent) {
                          ctx.close();
                        }
                      }
                    }
                );
              }
            }))
        ));
    return factory;
  }

}
```

完整项目可以参见：https://github.com/Gsealy/slowloris-webflux-netty

没有漏扫工具的可以试试：https://github.com/shekyan/slowhttptest

# 引用

1. [How to configure netty connection-timeout for Spring WebFlux Answer 1](https://stackoverflow.com/a/58195908/9137803)

