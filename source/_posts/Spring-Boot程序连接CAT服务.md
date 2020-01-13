---
title: Spring Boot程序连接CAT服务
tags:
  - Spring Boot
  - CAT
  - Spring Cloud
abbrlink: 6170f7da
date: 2018-10-11 13:49:04
---

CAT服务端部署完并启动之后，一步一步的将我们的Spring Boot应用连接上CAT。

demo 地址：

```shell
git clone https://github.com/Gsealy/CATClientDemo.git
```

### 一、构建Spring Boot应用

**方法一**：从[start.spring.io](https://start.spring.io/)快速构建完整项目，仅需添加`web`依赖。
![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/start-spring.jpg)

**方法二**：直接在IDE（idea或者STS）中创建Spring Boot应用即可。

先给出项目内文件结构：

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/spring-tree-map.jpg)


在`WebApplication`类中创建Controller，如下所示：

```java
@RestController
@SpringBootApplication
public class WebApplication {

  public static void main(String[] args) {
    SpringApplication.run(WebApplication.class, args);
  }

  /**
   * 监控访问id
   */
  @GetMapping("/index/{id}")
  public String index(@PathVariable int id) {
    System.out.println("id is : " + id);
    return "id: " + id;
  }
}
```

这样直接运行就是一个跑在8080端口上的web服务了。

在`pom.xml`中添加`cat-client`依赖，为了做单元测试，同时也添加一下junit依赖

```xml
<dependency>
  <groupId>com.dianping.cat</groupId>
  <artifactId>cat-client</artifactId>
  <version>2.0.0</version>
</dependency>
<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.12</version>
  <scope>test</scope>
</dependency>
```

### 二、添加配置文件

在`resources`目录下，新建`META-INF`文件夹

在`META-INF`中新建`app.properties`配置文件并添加应用名称，从而让CAT服务端能正常接收应用名称

```properties
app.name=webserver
```

### 三、添加埋点方案

从cat目录下框架埋点方案集成的文件夹中找到springboot的埋点方案，放进项目中。

```java
@Configuration
public class CatFilterConfigure {

  @Bean
  public FilterRegistrationBean<CatFilter> catFilter() {
    FilterRegistrationBean<CatFilter> registration = new FilterRegistrationBean<>();
    CatFilter filter = new CatFilter();
    registration.setFilter(filter);
    registration.addUrlPatterns("/*");
    registration.setName("cat-filter");
    registration.setOrder(1);
    return registration;
  }
}
```

### 四、修改Controller

修改之前的`index`方法，添加Transaction和Event。

```java
@GetMapping("/index/{id}")
  public String index(@PathVariable int id) {
    Transaction t = Cat
        .getProducer().newTransaction("URL", "Get.id");
    try {
      Cat.getProducer().newEvent("URL.Get", "id").setStatus(Message.SUCCESS);
      t.addData("id is:" + id);
      // 你的业务代码
      System.out.println("id is : " + id);

      t.setStatus(Transaction.SUCCESS);
    } catch (Exception e) {
      t.setStatus(e);
      throw e;
    } finally {
      t.complete();
    }
    return "id: " + id;
  }
```

### 五、运行测试

可以直接跑demo中的单元测试，也可以直接启动web服务，然后自测。

可以在CAT服务端看到当前的应用已经连上了

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/cat-with-spring.jpg)

在Transaction和Event中也都能看到相应的log。

结束！🔚

------

