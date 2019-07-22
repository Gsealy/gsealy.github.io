---
title: Route删除和Gateway-Filter优先级的小坑
tags:
  - Spring Cloud
  - Gateway
  - Filter
abbrlink: 6548debe
date: 2018-11-29 15:20:07
---

最近在做业务网关的时候，发现用`RouteLocatorBuilder`构建的`Route`无法操作删除和更新操作，又换回了`RouteDefinition`的构建方式，梳理下这里面的坑。

### Route不能删除

查看`GatewayControllerEndpoint`可以看到有这个删除的方法：

```java
	@DeleteMapping("/routes/{id}")
	public Mono<ResponseEntity<Object>> delete(@PathVariable String id) {
		return this.routeDefinitionWriter.delete(Mono.just(id))
				.then(Mono.defer(() -> Mono.just(ResponseEntity.ok().build())))
				.onErrorResume(t -> t instanceof NotFoundException, t -> Mono.just(ResponseEntity.notFound().build()));
	}
```

但是他只针对`RouteDefinition`方式构建的Route, 因为它会调用`delete()`这个在`RouteDefinitionWriter`接口中的方法，具体实现是在`InMemoryRouteDefinitionRepository`中，我们看最开始：

```java
public class InMemoryRouteDefinitionRepository implements RouteDefinitionRepository {

	private final Map<String, RouteDefinition> routes = synchronizedMap(new LinkedHashMap<String, RouteDefinition>());
    
    @Override
	public Mono<Void> save(Mono<RouteDefinition> route) {
		return route.flatMap( r -> {
			routes.put(r.getId(), r);
			return Mono.empty();
		});
	}
...
}
```

直接就是一个带`RouteDefinition`的Map，这也都是`save()`存进来的KV，可以很清楚的看到Key就是他的`Id`，然后对应着本身。所以删除的时候直接删除这一KV，然后再去`publisher`一下就行了。

反而`Route`就没有这好事，所以我不得不先放弃了这个，但是也提了issues [#675](https://github.com/spring-cloud/spring-cloud-gateway/issues/675), 看看官方怎么说吧。

### Factory不能实现Ordered

本想着构建`Factory`的时候，像普通的`GlobalFilter`或者`GatewayFilter`直接实现一个`Ordered`就能给他一个特定的优先级了。也不用改之前其他`Filters`定义好的优先级，但是不行啊！

我先给了个优先级，从官方给的`EndPoint`可以查看到：

![](https://ws1.sinaimg.cn/large/7074e5d2ly1fxozc8c4e5j20tg0ex78k.jpg)

但是这个设置了是不起作用的！只能`debug`看看了，进入到任意一个`GlobalFilter`，查看`GatewayFilterChain`的`filters`这一项，可以看到他的order是1！所以要改变的话，只要把应该在他后面的改的值比他大就行了。

![](https://ws1.sinaimg.cn/large/7074e5d2ly1fxozg99zabj20hs0dbq3t.jpg)

结束！🔚

------

