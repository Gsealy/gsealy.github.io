---
title: RocketMQ Idle控制导致频繁打印closeChannel问题排查
tags:
  - RocketMQ
abbrlink: 60b156cc
date: 2020-04-07 17:42:20
---

## 问题说明

公司内部测试环境RocketMQ经常会打印关闭连接日志，具体如下所示：

```
INFO closeChannel: close the connection to remote address[xxx] result: true
```

这是一个INFO级别的日志，我相信阿里最开始写的时候是想监控何时关闭了连接，所以把这个设为了`INFO`级别，但是因为我们内部使用时日志的吞吐量并不高，尤其是单独测试某一功能时，可能并没有审计日志入库操作。所以会出现连接被关闭的问题。

ps. 此问题不影响业务正常运行，就是看着恶心。

## 问题分析

排查方法如下：

1. 定位日志具体位置。
2. 尝试屏蔽此日志输出，来达到眼不见心不烦。
3. 尝试定位具体问题，若可以解决就解决该问题。

### 日志具体位置

经全局搜索，日志是由`org.apache.rocketmq.remoting.common.RemotingUtil#closeChannel`方法打印的，具体如下：

```java
public static void closeChannel(Channel channel) {
    final String addrRemote = RemotingHelper.parseChannelRemoteAddr(channel);
    channel.close().addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            log.info("closeChannel: close the connection to remote address[{}] result: {}", addrRemote,
               future.isSuccess());
        }
    });
}
```

断点打在上面的方法入口，等待Idle调用，具体调用链如下：

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/rocketmq/close_invoke_line.png)

可以看到是由`IdleStateHandler`触发的关闭连接操作。

其实针对Idle的猜测还是因为看了RocketMQ的日志，在`namesrv.log`日志文件内：

```
2020-04-07 10:59:02 INFO NSScheduledThread1 - configTable SIZE: 0
2020-04-07 11:06:15 WARN NettyServerCodecThread_3 - NETTY SERVER PIPELINE: IDLE exception [127.0.0.1:2414]
2020-04-07 11:06:15 INFO NettyServerNIOSelector_3_3 - closeChannel: close the connection to remote address[127.0.0.1:2414] result: true
2020-04-07 11:06:15 INFO NettyServerCodecThread_3 - NETTY SERVER PIPELINE: channelInactive, the channel[127.0.0.1:2414]
2020-04-07 11:06:15 INFO NettyServerCodecThread_3 - NETTY SERVER PIPELINE: channelUnregistered, the channel[127.0.0.1:2414]
2020-04-07 11:07:06 INFO NettyServerCodecThread_5 - NETTY SERVER PIPELINE: channelRegistered 127.0.0.1:2706
2020-04-07 11:07:06 INFO NettyServerCodecThread_5 - NETTY SERVER PIPELINE: channelActive, the channel[127.0.0.1:2706]
```

NameSrv因为Idle异常从而关闭了Channel。

本来还有一个疑问，就是Spring默认开的`INFO`级别的日志，发现就算执行到了，有的info日志也不会打印。发现客户端中可以修改日志的提供者（Provider）,就把提供者改为Slf4j。这样就可以在`application.properties`中通过`log.level`控制日志级别了。

通过修改`RocketmqRemoting`的Log级别为debug，可以看到具体的原因。也就找到了上面的调用链。

ps. 因为这个问题在Spring Binder下通过配置无法修改，所以可以通过关闭`RocketmqRemoting`日志解决。

### 问题定位

我们知道了是因为Idle监测导致的连接被关闭，所以查找在哪里注册的`IdleStateHandler`，因为这一功能是Netty提供的。在`org.apache.rocketmq.remoting.netty.NettyRemotingClient`类中的`start()`重载方法内，也就是Netty的启动方法 ：

```java
pipeline.addLast(
    defaultEventExecutorGroup,
    new NettyEncoder(),
    new NettyDecoder(),
    new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()), // pipeline增加IdleHandler
    new NettyConnectManageHandler(),
    new NettyClientHandler());
```



其会从`nettyClientConfig`中获取All Idle的空闲时间（All Idle为读写任意idle超时就会触发，详见`io.netty.handler.timeout.IdleStateEvent`），知道了是从哪里来设置的。那就来看看是哪里设置进去的。

`NettyClientConfig`这个类是在`org.apache.rocketmq.remoting.netty`包中，会在`org.apache.rocketmq.client.impl.factory.MQClientInstance#MQClientInstance()`中构建并使用，这个方法无法通过外部配置进行修改。方法内容具体如下所示：

```java
 public MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook) {
	this.clientConfig = clientConfig;
	this.instanceIndex = instanceIndex;
	this.nettyClientConfig = new NettyClientConfig(); // 构建NettyClientConfig对象
   	this.nettyClientConfig.setClientCallbackExecutorThreads(clientConfig.getClientCallbackExecutorThreads());
	this.nettyClientConfig.setUseTLS(clientConfig.isUseTLS());
	this.clientRemotingProcessor = new ClientRemotingProcessor(this);
	this.mQClientAPIImpl = new MQClientAPIImpl(this.nettyClientConfig, this.clientRemotingProcessor, rpcHook, clientConfig);
...
}
```

仅支持两项配置的修改，可以通过本地Merge，也可以通过提起PR进行修改。

### 问题修复

1. 暂时屏蔽

   通过屏蔽`RocketmqRemoting`的日志进行屏蔽，同时也会屏蔽该命名下的其他Log。

2. 一劳永逸

   - 可以直接把`NettyRemotingClient`中的Idle检测删除
   - 可以通过修改`ClientConfig`和`MQClientInstance`中对应部分，扩展该方法。同时针对Spring Boot，也可扩展相应的properties配置。


🔚

------



