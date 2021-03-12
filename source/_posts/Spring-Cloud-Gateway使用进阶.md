---
title: Spring Cloud Gateway使用进阶
tags:
  - Spring Cloud
  - Gateway
abbrlink: 71acf729
date: 2019-11-18 16:35:07
---

## 前言

Spring Cloud Gateway（下称SCG）涉及几个最近使用上的问题，自己感觉还挺典型的。所以罗列出来。

- 缓存请求体会二次过滤
- 服务端响应头删除

当前环境版本：

- Spring Boot 2.1.6.RELEASE
- Spring Cloud Greenwich.SR3
  - 对应SCG版本2.1.3.RELEASE

## 缓存请求体会二次过滤

> 这是一个bug，官方已经在2019.10.2修复，但是因为本地环境所限，无法直接升级版本

SCG就缓存请求体这一需求，社区很早就已经给出了解决方案，但是方案不是很完美。所以经过了几次的修改，我自己使用上，觉得现在的版本用着还可以。

### 问题复现

工程详见：[error-cache-body](https://gsealy.coding.net/public/error-cache-body/error-cache-body/git/files)

直接下载运行，post请求`:8080/post`即可，执行：

```powershell
http post :8080/post foo=bar
```

Log如下：

```
2019-11-18 18:07:32.899 TRACE 64684 --- [ctor-http-nio-2] o.s.c.g.f.WeightCalculatorWebFilter      : Weights attr: {}
2019-11-18 18:07:32.913 DEBUG 64684 --- [ctor-http-nio-2] o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition httpbin_route applying {_genkey_0=/post} to Path
2019-11-18 18:07:32.914 DEBUG 64684 --- [ctor-http-nio-2] o.s.c.g.r.RouteDefinitionRouteLocator    : RouteDefinition matched: httpbin_route
2019-11-18 18:07:32.921 TRACE 64684 --- [ctor-http-nio-2] o.s.c.g.h.p.RoutePredicateFactory        : Pattern "/post" matches against value "/post"
2019-11-18 18:07:32.922 DEBUG 64684 --- [ctor-http-nio-2] o.s.c.g.h.RoutePredicateHandlerMapping   : Route matched: httpbin_route
2019-11-18 18:07:32.922 DEBUG 64684 --- [ctor-http-nio-2] o.s.c.g.h.RoutePredicateHandlerMapping   : Mapping [Exchange: POST http://localhost:8080/post] to Route{id='httpbin_route', uri=http://httpbin.org:80, order=0, predicate=Paths: [/post], match trailing slash: true, gatewayFilters=[]}
2019-11-18 18:07:32.922 DEBUG 64684 --- [ctor-http-nio-2] o.s.c.g.h.RoutePredicateHandlerMapping   : [0a25a24b] Mapped to org.springframework.cloud.gateway.handler.FilteringWebHandler@54bb8e8f
2019-11-18 18:07:32.924 DEBUG 64684 --- [ctor-http-nio-2] o.s.c.g.handler.FilteringWebHandler      : Sorted gatewayFilterFactories: [[GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.RemoveCachedBodyFilter@5a08efdc}, order = -2147483648], [GatewayFilterAdapter{delegate=io.gsealy.cachebody.filter.CacheRequestGlobalFilter@625dfff3}, order = -2147482648], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.AdaptCachedBodyGlobalFilter@72fd8a3c}, order = -2147482648], [GatewayFilterAdapter{delegate=io.gsealy.cachebody.filter.AccessLogGlobalFilter@5cf3157b}, order = -2147463648], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.NettyWriteResponseFilter@1e9469b8}, order = -1], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.ForwardPathFilter@648d0e6d}, order = 0], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter@57272109}, order = 10000], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.config.GatewayNoLoadBalancerClientAutoConfiguration$NoLoadBalancerClientFilter@17273273}, order = 10100], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.WebsocketRoutingFilter@79e66b2f}, order = 2147483646], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.NettyRoutingFilter@26350ea2}, order = 2147483647], [GatewayFilterAdapter{delegate=org.springframework.cloud.gateway.filter.ForwardRoutingFilter@59696551}, order = 2147483647]]
2019-11-18 18:07:32.942 TRACE 64684 --- [ctor-http-nio-2] o.s.c.g.support.ServerWebExchangeUtils   : retaining body in exchange attribute
2019-11-18 18:07:32.944 TRACE 64684 --- [ctor-http-nio-2] i.g.c.filter.AccessLogGlobalFilter       : access 1 times.
2019-11-18 18:07:32.946 TRACE 64684 --- [ctor-http-nio-2] o.s.c.g.filter.RouteToRequestUrlFilter   : RouteToRequestUrlFilter start
2019-11-18 18:07:33.246 TRACE 64684 --- [ctor-http-nio-6] o.s.c.gateway.filter.NettyRoutingFilter  : outbound route: 6867371b, inbound: [0a25a24b] 
2019-11-18 18:07:34.265 TRACE 64684 --- [ctor-http-nio-6] o.s.c.g.filter.NettyWriteResponseFilter  : NettyWriteResponseFilter start inbound: 6867371b, outbound: [0a25a24b] 
2019-11-18 18:07:34.276 TRACE 64684 --- [ctor-http-nio-2] i.g.c.filter.AccessLogGlobalFilter       : access 2 times.
2019-11-18 18:07:34.276 TRACE 64684 --- [ctor-http-nio-2] o.s.c.g.filter.RouteToRequestUrlFilter   : RouteToRequestUrlFilter start
2019-11-18 18:07:34.277 TRACE 64684 --- [ctor-http-nio-2] o.s.c.g.filter.NettyWriteResponseFilter  : NettyWriteResponseFilter start inbound: 6867371b, outbound: [0a25a24b] 
2019-11-18 18:07:34.279 TRACE 64684 --- [ctor-http-nio-6] o.s.c.g.filter.RemoveCachedBodyFilter    : releasing cached body in exchange attribute
```

上面的是一个完整的请求，开启了`org.springframework.cloud.gateway`的`trace`日志。其中可以看到`AccessLogGlobalFilter`访问了两次，提取出来，如下所示：

```
i.g.c.filter.AccessLogGlobalFilter       : access 1 times.
i.g.c.filter.AccessLogGlobalFilter       : access 2 times.
```

第二次在请求在走filter的时候，其实第一次的请求已经向后端提交了，所以二次请求无效。对响应内容也没有影响，二次请求不会打印`retaining body in exchange attribute`，所以可以断定重新过的时候，body被删除或者清空了，所以没有缓存的操作，其他的请求内容就会使用一些**不可变**对象封装请求，例如：请求头会使用`org.springframework.http.ReadOnlyHttpHeaders`对象封装，无法操作请求Header。

相关Issues：[gh-1315](https://github.com/spring-cloud/spring-cloud-gateway/issues/1315)，PR：[ 856a2417b6535d54d8f07625dfb48bc5080e87fe ]( https://github.com/spring-cloud/spring-cloud-gateway/commit/856a2417b6535d54d8f07625dfb48bc5080e87fe)

### 问题解析

官方的PR解决了整个二次请求的问题，`This allows the AdaptCahcedBodyGlobalFilter to not worry about handling empty.`现在就可以处理空请求体了。可以直接升级到`2.2.0.RC2`版本来生效，或者直接把相关代码拷贝到当前实现类中即可。下面这个`BodyUtils`可以拆掉请求体外部封装并转为Map或String等形式，供后面Filter使用。

包含如下方法：

- BodyUtils#toRaw(DataBuffer body) - 获取body内的String
- BodyUtils#toFormDataMap(ServerHttpRequest httpRequest, DataBuffer body) - body转为FormData, 针对content-type为`application/x-www-form-urlencoded`
- BodyUtils#cacheRequestBody - 缓存Body，ServerWebExchangeUtils中的完整拷贝

**NOTE：**Reactive Streams中不推荐做这种转换

地址：[BodyUtils Gist](https://gist.github.com/ccb1fe95e03fd854deb53fd4f474d5db)

## 服务端响应头删除

在用`httpbin.org`做报文请求响应测试时，服务端会添加一系列的响应头，比如一些安全请求头，但是网关本身就开了`SecureHeader`的Filter，会出现重复

### 问题复现

当Body缓存没有修复时，执行不到这一步就会抛出异常，因为二次请求会报针对只读请求头的写操作异常（UnsupportedOperationException），当前问题和Body缓存也无关，可以抛开缓存来看。

工程详见：[multi-response-header](https://gsealy.coding.net/public/multi-response-header/multi-response-header/git)

所有配置都在`application.properties`中，包括：

- 安全头配置 - 全局配置，只开启`x-frame-options`，值为`SAMEORIGIN`
- 转发配置 - 转发至httpbin.org

```properties
spring.cloud.gateway.filter.secure-headers.frame-options=SAMEORIGIN
spring.cloud.gateway.filter.secure-headers.disable=x-xss-protection,strict-transport-security,x-content-type-options,referrer-policy,content-security-policy,x-download-options,x-permitted-cross-domain-policies
spring.cloud.gateway.default-filters=SecureHeaders
```

执行如下请求：

```powershell
λ http post :8080/post -v
POST /post HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Content-Length: 0
Host: localhost:8080
User-Agent: HTTPie/1.0.3

HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Content-Encoding: gzip
Content-Length: 281
Content-Type: application/json
Date: Wed, 20 Nov 2019 09:11:38 GMT
Referrer-Policy: no-referrer-when-downgrade
Server: nginx
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
```

可以看上面的响应头，有两个`X-Frame-Options`安全头，`SAMEORIGIN`是网关添加的，而`DENY`是httpbin.org响应的请求头。（可以做一个直接请求httpbin的作为对比）

### 问题解析

这里我们以当前最新的`2.1.4.RELEASE`版本的SCG为例，`SecureHeadersGatewayFilterFactory`会在响应前把所需的返回客户端的头写进Exchange的Response中，这不影响Request的发送。但是在处理响应时，会将预先的设置好的头放到完整响应中，具体在`NettyRoutingFilter.java`的[118行]( https://github.com/spring-cloud/spring-cloud-gateway/blob/2f7fc76933a6da43048ac94a39a3db83e417ac6e/spring-cloud-gateway-core/src/main/java/org/springframework/cloud/gateway/filter/NettyRoutingFilter.java#L188 )，如下：

```java
// NettyRoutingFilter.java

ServerHttpResponse response = exchange.getResponse();
.....
if (!filteredResponseHeaders.containsKey(HttpHeaders.TRANSFER_ENCODING)
	&& filteredResponseHeaders.containsKey(HttpHeaders.CONTENT_LENGTH)) {
	response.getHeaders().remove(HttpHeaders.TRANSFER_ENCODING);
}

exchange.getAttributes().put(CLIENT_RESPONSE_HEADER_NAMES,
	filteredResponseHeaders.keySet());

// 响应头整合
response.getHeaders().putAll(filteredResponseHeaders);

return Mono.just(res);
.....
```

在整合之前，会有一个请求头过滤的操作，包含转发头的过滤等，为了去掉后端发回来的相同头，需要先删除响应中的头。

```java
// NettyRoutingFilter.java
// 响应头过滤操作

// make sure headers filters run after setting status so it is
// available in response
HttpHeaders filteredResponseHeaders = HttpHeadersFilter
	.filter(getHeadersFilters(), headers, exchange, Type.RESPONSE);
```

SCG提供了`HttpHeadersFilter`，用于过滤请求/响应头，先解析一下这个类：

```java
public interface HttpHeadersFilter {

    /**
    * 针对下面的filter方法的封装，主要用于过滤Request请求
    */
	static HttpHeaders filterRequest(List<HttpHeadersFilter> filters,
			ServerWebExchange exchange) {
		HttpHeaders headers = exchange.getRequest().getHeaders();
		return filter(filters, headers, exchange, Type.REQUEST);
	}

    /**
    * 使用java stream的filter做过滤
    *
    * @param filters header filter集合
    * @param input 需要过滤的headers
    * @param exchange ServerWebExchange
    * @param type 请求/响应，定义在内部枚举类
    */
	static HttpHeaders filter(List<HttpHeadersFilter> filters, HttpHeaders input,
			ServerWebExchange exchange, Type type) {
		HttpHeaders response = input;
		if (filters != null) {
			HttpHeaders reduce = filters.stream()
					.filter(headersFilter -> headersFilter.supports(type)).reduce(input,
							(headers, filter) -> filter.filter(headers, exchange),
							(httpHeaders, httpHeaders2) -> {
								httpHeaders.addAll(httpHeaders2);
								return httpHeaders;
							});
			return reduce;
		}

		return response;
	}

	/**
	 * 需要实现的自定义逻辑
	 * Filters a set of Http Headers.
	 * @param input Http Headers
	 * @param exchange a {@link ServerWebExchange} that should be filtered
	 * @return filtered Http Headers
	 */
	HttpHeaders filter(HttpHeaders input, ServerWebExchange exchange);

	default boolean supports(Type type) {
		return type.equals(Type.REQUEST);
	}
    
	enum Type {
		REQUEST, RESPONSE
	}
}
```

实现一个`HttpHeadersFilter`，加上自定义逻辑就可以了。ps. 如果要操作响应的话，记得实现一下default方法support()，针对安全头过滤的实现如下：

```java
/**
 * 删除上游响应的安全请求头, 添加网关默认安全请求头
 *
 * @author Gsealy
 */
@Component
@Slf4j
public class RemoveSecureHeaderFilter implements HttpHeadersFilter, Ordered {

    // TODO: 可以添加需要删除的安全请求头, 这里只删除X-Frame-Options
    private final Set<String> secureHeaders = new HashSet<>(Arrays.asList("x-frame-options"));

    @Override
    public HttpHeaders filter(HttpHeaders input, ServerWebExchange exchange) {
        HttpHeaders filtered = new HttpHeaders();
        input.entrySet().stream()
                .filter(entry -> !this.secureHeaders.contains(entry.getKey().toLowerCase()))
                .forEach(entry -> filtered.addAll(entry.getKey(), entry.getValue()));
        if (log.isTraceEnabled()) {
            log.trace("filtered headers: {}", filtered);
        }
        return filtered;
    }

    @Override
    public boolean supports(Type type) {
        return type.equals(Type.RESPONSE);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

结束！🔚

------