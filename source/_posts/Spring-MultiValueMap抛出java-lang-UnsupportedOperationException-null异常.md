---
title: 'Spring MultiValueMap抛出java.lang.UnsupportedOperationException: null异常'
date: 2018-11-19 17:54:23
tags:
- Spring Cloud
- Gateway
---

今儿在sc-Gateway处理formData的时候，明明有值，但是会抛出`java.lang.UnsupportedOperationException: null`的异常。

看一下代码：

```java
if (delegate.getMethod().matches("POST")) {
	if (StringUtils.equals(mediatype.toString(), MediaType.APPLICATION_FORM_URLENCODED_VALUE)) {
        Mono<MultiValueMap<String, String>> formdata = exchange.getFormData();
        addFormDataToMap(formdata, paramsMap);
      } else {
          ...
      }
}
```

Spring Cloud Gateway是建在Spring 5和Spring Boot 2.0之上的。所以是响应式的编程，上手比较困难。这里是做一个FormData的转换，将键值对形式的FormData加载到Query中去，再继续做下一步操作。

`addFormDataToMap`方法是这样的：

```java
public void addFormDataToMap(Mono<MultiValueMap<String, String>> formdata,
      MultiValueMap<String, String> paramsMap) {
    AtomicReference<MultiValueMap<String, String>> QueryRef = new AtomicReference<>();
    formdata.subscribe(maps -> {
      QueryRef.set(maps);
    });
    if (QueryRef.get().isEmpty() && QueryRef.get() == null) {
      return;
    }
    paramsMap.addAll(QueryRef.get()); // <1>
  }
```

`paramsMap`存储的是Query中的键值对，同过formdata的subscribe，存储进`QueryRef`中。最后在`addAll`进`paramsMap`。

这样看起来没什么问题，但是当执行`<1>`时，会抛出异常：`java.lang.UnsupportedOperationException: null`

跟进去发现，在`CollectionUtils`类中，可以看到存在完整的键值对，执行完当前step会直接抛出异常。

![](https://ws1.sinaimg.cn/large/7074e5d2ly1fxe95vjgvlj20ob034glq.jpg)

subscribe抛出异常：

![](https://ws1.sinaimg.cn/large/7074e5d2ly1fxe98g33xuj20q009faan.jpg)

查看`java.util.Map`方法中的`computeIfAbsent`类，当当前Map不支持此`put`操作时抛出`UnsupportedOperationException`异常，所以存储了一个`null`

![](https://ws1.sinaimg.cn/large/7074e5d2ly1fxe9ddhvvhj20o107gdgl.jpg)

修改代码：

```java
public void addFormDataToMap(Mono<MultiValueMap<String, String>> formdata, MultiValueMap<String, String> paramsMap) {
	AtomicReference<MultiValueMap<String, String>> QueryRef = new AtomicReference<>();
    formdata.subscribe(maps -> {
      QueryRef.set(maps);
    });
    if (QueryRef.get().isEmpty() && QueryRef.get() == null) {
      return;
    }
    LinkedMultiValueMap<String, String> newList = new LinkedMultiValueMap<>(paramsMap); // {1-1}
    newList.addAll(QueryRef.get()); // {1-2}
}
```

删除原先代码`<1>`部分，增加`{1-1}`和`{1-2}`，先创建新的`LinkedList`，再`addAll`

> #### 引用
>
> [Why do I get an UnsupportedOperationException when trying to remove an element from a List?](https://stackoverflow.com/a/2965762)🔚

------

