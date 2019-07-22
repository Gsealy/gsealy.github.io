---
title: Spring 5 WebFlux使用Swagger2作为API管理工具
tags:
  - WebFlux
  - Swagger
abbrlink: d1dfb98a
date: 2019-03-19 09:56:17
---

### 前言

找了一圈， 大家在Spring boot 2搭配Spring 5使用的时候。还是使用的Spring MVC的形式，没有直接上WebFlux的形式。正巧看到Springfox也发了快照版本支持WebFlux，今儿我就把这些整理一下。

> 使用工具和版本：
>
> IntelliJ IDEA 2018.3.5 (Ultimate Edition)
>
> JDK 1.8
>
> Windows 10
>
> Spring boot: 2.1.3.RELEASE
>
> Springfox: 3.0.0.SNAPSHOT

### 上手

可以直接去[start.spring.io](https://start.spring.io)快速构建一个程序，也可以看我的Github：[link](https://github.com/Gsealy/some-webflux-extensions)

#### 一、POM

不去概述`Swagger`的作用了，直接上手。先在pom中引入以下依赖（需要先添加下Springfox快照所在的maven库）

```xml
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger-ui</artifactId>
      <version>3.0.0-SNAPSHOT</version>
    </dependency>

    <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger2</artifactId>
      <version>3.0.0-SNAPSHOT</version>
    </dependency>

    <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-spring-webflux</artifactId>
      <version>3.0.0-SNAPSHOT</version>
    </dependency>
  </dependencies>
  <repositories>
    <repository>
      <id>oss-snapshot</id>
      <name>OSS Snapshot</name>
      <url>http://oss.jfrog.org/oss-snapshot-local</url>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </repository>
  </repositories>
```

全部都使用3.0.0的快照版本，因为只有在当前版本支持了WebFlux，包含`@EnableSwagger2WebFlux`的注解。

#### 二、配置

启用swagger，在main方法中添加`@EnableSwagger2WebFlux`

```java
@EnableSwagger2WebFlux
@SpringBootApplication
public class Boot2WithSwaggerApplication {

  public static void main(String[] args) {
    SpringApplication.run(Boot2WithSwaggerApplication.class, args);
  }

}
```

**Configuration**

创建一个`Docket`bean来配置Swagger，一个`Docket`实例为API配置提供默认设置和便捷的配置方法。

```java
@Configuration
public class swaggerConfiguration {

  @Bean
  public Docket swaggerApi() {
    return new Docket(DocumentationType.SWAGGER_2).apiInfo(swaggerApiInfo()).select() 
 .apis(RequestHandlerSelectors.basePackage("io.github.gsealy.boot2withswagger.controller")) 
        .paths(PathSelectors.any())
        .build();
  }

  private ApiInfo swaggerApiInfo() {
    return new ApiInfoBuilder().title("webflux-swagger2 API doc")
        .description("how to use this")
        .termsOfServiceUrl("https://github.com/Gsealy")
        .contact(new Contact("Gsealy", "https://gsealy.github.io", "gsealy@gmail.com")) 
        .version("1.0")
        .build();
  }
}
```

**Controller**

在Controller上就可以直接使用Swagger提供的注解了，但是不清楚现在是不是支持Webflux的Handler写法。

下面给了一个包含主要四个HTTP方法的Controller示例

```java
@RestController
@RequestMapping("/apis")
@Api(value = "Swagger test Controller", description = "learn how to use swagger")
public class SwaggerController {

  @GetMapping
  @ApiOperation(value = "GET Method", response = String.class)
  public Mono<String> get() {
    return Mono.just("this is GET Met" + "hod.");
  }

  @PostMapping
  @ApiOperation(value = "POST Method", response = String.class)
  public Mono<String> post() {
    return Mono.just("this is POST Method.");
  }

  @PutMapping
  @ApiOperation(value = "PUT Method", response = String.class)
  public Mono<String> put() {
    return Mono.just("this is PUT Method.");
  }

  @DeleteMapping
  @ApiOperation(value = "DELETE Method", response = String.class)
  public Mono<String> delete() {
    return Mono.just("this is DELETE Method.");
  }

}
```

### 视图

启动程序后，访问`http://localhost:8080/swagger-ui.html/`，看下页面显示

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/swagger-ui.png)

就是标准的Swagger-ui页面了，具体swagger使用方法可以去官网看看doc

### 引用

[Spring Boot 2 RESTful API Documentation With Swagger 2 Tutorial](https://dzone.com/articles/spring-boot-2-restful-api-documentation-with-swagg)

[Springfox Reference Documentation](http://springfox.github.io/springfox/docs/snapshot/)

结束！🔚

------

