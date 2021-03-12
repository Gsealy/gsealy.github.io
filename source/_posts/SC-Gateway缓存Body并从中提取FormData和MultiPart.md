---
title: SC-Gateway缓存Body并从中提取FormData和MultiPart
tags:
  - Gateway
  - Spring Cloud
  - WebFlux
abbrlink: 635b6c82
date: 2018-11-27 14:11:45
---

> 更新于 2021年3月12日
>
> 新增具体实例，感谢yvanbaker指出的Part文件上传读取问题

# 示例

通过创建一个`XSS过滤`的全局过滤器，来展示form-data，part，及url的过滤操作。

## 缓存Body

在SCG 2.1.4+，支持body题缓存为`DataBuffer`，去掉了外面的`Flux`包装，具体可以参考[ServerWebExchangeUtils#cacheRequestBody](https://github.com/spring-cloud/spring-cloud-gateway/blob/eb9611a58a46a50a3a330c213a13965854c52deb/spring-cloud-gateway-server/src/main/java/org/springframework/cloud/gateway/support/ServerWebExchangeUtils.java#L332)

那下面就用2.1.1的SCG来做，把body缓存拿到低版本来试试。

新建一个`GlobalFilter`做缓存，把所有操作都封装到`BodyUtils.cacheRequestBody`

```java
return BodyUtils.cacheRequestBody(exchange, (serverHttpRequest) -> {
    if (serverHttpRequest == exchange.getRequest()) {
        return chain.filter(exchange);
    }
    return chain.filter(exchange.mutate().request(serverHttpRequest).build());
});
```

## XSS过滤

1. 先从Attribute中取到缓存的`Databuffer`
2. 分别针对header，query，form-data，multipart做过滤操作
3. 重新封装请求，返回给过滤链（Filter Chain）

具体内容可以查看demo：[escape-request](https://gsealy.coding.net/public/escape-request/escape-request/git/files)

----

# 以下为旧内容，不推荐

### 缓存Body

缓存Body的话， Gateway给出了一个工厂类，可以直接用。也可以像我有别的需求的，重写一下。

我主要就给他改成`GlobalFilter`，然后给了最高优先级，就想在入口处就拿到缓存内容，放到Attribute中，以供后面使用。

给一个Gist的链接吧 : [Link](https://gist.github.com/Gsealy/a468da7d36a65de9f5b959f2b20f0bca)

### 提取数据

困扰我好久的问题主要就是`FormData`和`Part`的使用问题，因为需要做参数映射、Hash计算等操作，需要用到这些body中的内容，也不能直接拿Body去计算，还是需要转换为标准的Data格式，比如`FormData`主要就是键值对，`MultiPart`主要就是上传文件，然后给他转成输入流`InputStream`.

#### Body转换

本来还提了一个issues [#671](https://github.com/spring-cloud/spring-cloud-gateway/issues/671) , 但是因为没有给完整demo而是给了一个Gist就被关了(笑，确实是自己懒省事了). 但是今儿在rebuild demo的时候想到，既然我都已经拿到了Body，直接转换一下不就行了么，也解决了`Mono`形式的`FormData`被消费，后端服务无法接收的问题。

当我发现了[BodyExtractors](https://github.com/spring-projects/spring-framework/blob/a5339d71eae50e2cb6e572d52a823a26d1d103f1/spring-webflux/src/main/java/org/springframework/web/reactive/function/BodyExtractors.java)类, 其中有`toFormData()`和`toMultipartData()`方法, 结合测试类[BodyExtractorsTests](https://github.com/spring-projects/spring-framework/blob/cfb1ed1009bebe7d7fbb10908dbdbf3bae934548/spring-webflux/src/test/java/org/springframework/web/reactive/function/BodyExtractorsTests.java) 就可以实现提取了

Gist : [Link](https://gist.github.com/Gsealy/1450a0df67a80d56b6b430627af5a4ac)

完整Demo-FormData ：[zip](https://github.com/Gsealy/SignatureTest/raw/master/datademo.zip)

结束！🔚

------

