---
title: 本地构建基于Alpine的OpenJ9 OpenJDK11及精简JDK的制作
tags:
  - Docker
  - Java
  - OpenJ9
abbrlink: '147e3088'
date: 2020-05-08 16:20:16
---

看说OpenJ9的内存占用相较于Hotspot要低，[对比文章](https://billykorando.com/2019/05/03/5-reasons-why-you-should-consider-switching-to-eclipse-openj9/)。所以打算在本地打一个镜像试试。

其实AdoptOpenJDK提供的有OpenJ9的镜像，[链接在此](https://hub.docker.com/r/adoptopenjdk/openjdk11-openj9)。因为需要做一些修改，所以直接用Dockerfile本地构建了。同时也分享一下是如何在服务器无法连接代理的情况下使用AdoptOpenJDK镜像脚本构建镜像。

# 前期准备

参考的构建脚本是AdoptOpenJDK的openjdk-docker项目，具体[链接](https://github.com/AdoptOpenJDK/openjdk-docker/blob/master/11/jre/alpine/Dockerfile.openj9.releases.full)。具体内容就不放这里了，直接去链接里面看就好了。

1. 为什么构建脚本那么长
   
   OpenJ9现在基于Alpine的好像只能用glibc库，但是Alpine官方用的是musl-libc，所以脚本中很大一部分都是在安装glibc，具体可以看：[Installing openjdk 11 on alpine:3.9](https://serverfault.com/a/960783)

2. 国内访问问题
   
   脚本中有很多的curl操作，而且地址对应的是github和archlinux，github用的AWS云存储，国内无法直连，archlinux的archive直连下载只有20kb左右，我搜了下国内的镜像站点都没有收录，所以直接在服务器上构建不可取，需要稍微修改下。

## 修改脚本

1. 下载文件
   
   先在本地通过代理将所需文件下载至本地，本地搭建一个文件服务器供服务器使用（我这里所用的服务器和工作机是相同网段）。下面也给出了所需的文件，都放在天翼云上了， 国内下载应该是满速。
   
   本地依赖下载：[天翼云盘](https://cloud.189.cn/web/share?code=Yn6Fju7Vb2qq)（访问码：rbp5）

2. 搭建文件服务器
   
   文件服务器很简单，有一个基于Chrome的插件可以直接开启服务。[Web Server for Chrome](https://chrome.google.com/webstore/detail/web-server-for-chrome/ofhbbkphhbklhfoeikjpcbhemlocgigb)
   
   操作很简单，选择文件目录，开启服务就好了，确保服务器能够正常访问即可。

3. 修改URL
   
   在上下两个`RUN`语法糖中都加入`URL_PREFIX="http://10.20.61.27:8887"`（完整Dockerfile见最后），修改所有出现的URL就可以了！别忘记添加Alpine CDN的国内镜像源。

# 构建JDK

脚本修改好之后，直接在服务端构建即可。

```shell
docker build . -t openjdk:11.0-openj9
```

```shell
Sending build context to Docker daemon  503.8kB
Step 1/9 : FROM alpine:3.11
 ---> f70734b6a266
Step 2/9 : MAINTAINER Gsealy <jiaojingwei@infosec.com.cn>
 ---> Running in 8750fff47cb4
Removing intermediate container 8750fff47cb4
 ---> 2b9b4628a331
Step 3/9 : ENV LANG="zh_CN.UTF-8" LC_ALL="zh_CN.UTF-8"
 ---> Running in 370ad5d94d09
Removing intermediate container 370ad5d94d09
 ---> 5384261f7b05
Step 4/9 : RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories     && apk add --no-cache --virtual .build-deps curl binutils     && GLIBC_VER="2.31-r0"     && URL_PREFIX="http://10.20.61.27:8887"     && GCC_LIBS_URL="${URL_PREFIX}/gcc-libs-9.1.0-2-x86_64.pkg.tar.xz"     && ZLIB_URL="${URL_PREFIX}/zlib-1_1.2.11-3-x86_64.pkg.tar.xz"     && curl -LfsS ${URL_PREFIX}/sgerrand.rsa.pub -o /etc/apk/keys/sgerrand.rsa.pub     && curl -LfsS ${URL_PREFIX}/glibc-${GLIBC_VER}.apk > /tmp/glibc-${GLIBC_VER}.apk     && apk add --no-cache /tmp/glibc-${GLIBC_VER}.apk     && curl -LfsS ${URL_PREFIX}/glibc-bin-${GLIBC_VER}.apk > /tmp/glibc-bin-${GLIBC_VER}.apk     && apk add --no-cache /tmp/glibc-bin-${GLIBC_VER}.apk     && curl -Ls ${URL_PREFIX}/glibc-i18n-${GLIBC_VER}.apk > /tmp/glibc-i18n-${GLIBC_VER}.apk     && apk add --no-cache /tmp/glibc-i18n-${GLIBC_VER}.apk     && /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 "$LANG" || true     && echo "export LANG=$LANG" > /etc/profile.d/locale.sh     && curl -LfsS ${GCC_LIBS_URL} -o /tmp/gcc-libs.tar.xz     && mkdir /tmp/gcc     && tar -xf /tmp/gcc-libs.tar.xz -C /tmp/gcc     && mv /tmp/gcc/usr/lib/libgcc* /tmp/gcc/usr/lib/libstdc++* /usr/glibc-compat/lib     && strip /usr/glibc-compat/lib/libgcc_s.so.* /usr/glibc-compat/lib/libstdc++.so*     && curl -LfsS ${ZLIB_URL} -o /tmp/libz.tar.xz     && mkdir /tmp/libz     && tar -xf /tmp/libz.tar.xz -C /tmp/libz     && mv /tmp/libz/usr/lib/libz.so* /usr/glibc-compat/lib     && apk del --purge .build-deps glibc-i18n     && rm -rf /tmp/*.apk /tmp/gcc /tmp/gcc-libs.tar.xz /tmp/libz /tmp/libz.tar.xz /var/cache/apk/*
 ---> Running in f702ac6a4a05
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/community/x86_64/APKINDEX.tar.gz
(1/8) Installing ca-certificates (20191127-r1)
(2/8) Installing nghttp2-libs (1.40.0-r0)
(3/8) Installing libcurl (7.67.0-r0)
(4/8) Installing curl (7.67.0-r0)
(5/8) Installing libgcc (9.2.0-r4)
(6/8) Installing libstdc++ (9.2.0-r4)
(7/8) Installing binutils (2.33.1-r0)
(8/8) Installing .build-deps (20200509.020904)
Executing busybox-1.31.1-r9.trigger
Executing ca-certificates-20191127-r1.trigger
OK: 18 MiB in 22 packages
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/community/x86_64/APKINDEX.tar.gz
(1/1) Installing glibc (2.31-r0)
OK: 27 MiB in 23 packages
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/community/x86_64/APKINDEX.tar.gz
(1/1) Installing glibc-bin (2.31-r0)
Executing glibc-bin-2.31-r0.trigger
/usr/glibc-compat/sbin/ldconfig: /usr/glibc-compat/lib/ld-linux-x86-64.so.2 is not a symbolic link

OK: 30 MiB in 24 packages
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/community/x86_64/APKINDEX.tar.gz
(1/1) Installing glibc-i18n (2.31-r0)
OK: 55 MiB in 25 packages
failed to set locale!
[warning] No definition for LC_PAPER category found
failed to set locale!
[warning] No definition for LC_NAME category found
failed to set locale!
[warning] No definition for LC_ADDRESS category found
failed to set locale!
[warning] No definition for LC_TELEPHONE category found
failed to set locale!
[warning] No definition for LC_MEASUREMENT category found
failed to set locale!
[warning] No definition for LC_IDENTIFICATION category found
WARNING: Ignoring APKINDEX.5f57d7a5.tar.gz: No such file or directory
WARNING: Ignoring APKINDEX.e1c8646f.tar.gz: No such file or directory
(1/8) Purging .build-deps (20200509.020904)
(2/8) Purging curl (7.67.0-r0)
(3/8) Purging binutils (2.33.1-r0)
(4/8) Purging glibc-i18n (2.31-r0)
(5/8) Purging libcurl (7.67.0-r0)
(6/8) Purging ca-certificates (20191127-r1)
Executing ca-certificates-20191127-r1.post-deinstall
(7/8) Purging nghttp2-libs (1.40.0-r0)
(8/8) Purging libstdc++ (9.2.0-r4)
Executing busybox-1.31.1-r9.trigger
Executing glibc-bin-2.31-r0.trigger
/usr/glibc-compat/sbin/ldconfig: /usr/glibc-compat/lib/ld-linux-x86-64.so.2 is not a symbolic link

OK: 17 MiB in 17 packages
Removing intermediate container f702ac6a4a05
 ---> 06ea75045e4f
Step 5/9 : ENV JAVA_VERSION jdk-11.0.7+10_openj9-0.20.0
 ---> Running in 1b52218826fb
Removing intermediate container 1b52218826fb
 ---> 10ac97e30305
Step 6/9 : RUN set -eux;     sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories;     URL_PREFIX="http://10.20.61.27:8887";     apk add --no-cache --virtual .fetch-deps curl;     curl -LfsSo /tmp/openjdk.tar.gz ${URL_PREFIX}/OpenJDK11U-jdk_x64_linux_openj9_11.0.7_10_openj9-0.20.0.tar.gz ;     mkdir -p /opt/java/openjdk;     cd /opt/java/openjdk;     tar -xf /tmp/openjdk.tar.gz --strip-components=1;     apk del --purge .fetch-deps;     rm -rf /var/cache/apk/*;     rm -rf /tmp/openjdk.tar.gz;
 ---> Running in 5a861f325886
+ sed -i s/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g /etc/apk/repositories
+ URL_PREFIX=http://10.20.61.27:8887
+ apk add --no-cache --virtual .fetch-deps curl
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/main/x86_64/APKINDEX.tar.gz
fetch http://mirrors.ustc.edu.cn/alpine/v3.11/community/x86_64/APKINDEX.tar.gz
(1/5) Installing ca-certificates (20191127-r1)
(2/5) Installing nghttp2-libs (1.40.0-r0)
(3/5) Installing libcurl (7.67.0-r0)
(4/5) Installing curl (7.67.0-r0)
(5/5) Installing .fetch-deps (20200509.020918)
Executing busybox-1.31.1-r9.trigger
Executing ca-certificates-20191127-r1.trigger
Executing glibc-bin-2.31-r0.trigger
/usr/glibc-compat/sbin/ldconfig: /usr/glibc-compat/lib/ld-linux-x86-64.so.2 is not a symbolic link

OK: 18 MiB in 22 packages
+ curl -LfsSo /tmp/openjdk.tar.gz http://10.20.61.27:8887/OpenJDK11U-jdk_x64_linux_openj9_11.0.7_10_openj9-0.20.0.tar.gz
+ mkdir -p /opt/java/openjdk
+ cd /opt/java/openjdk
+ tar -xf /tmp/openjdk.tar.gz '--strip-components=1'
+ apk del --purge .fetch-deps
WARNING: Ignoring APKINDEX.5f57d7a5.tar.gz: No such file or directory
WARNING: Ignoring APKINDEX.e1c8646f.tar.gz: No such file or directory
(1/5) Purging .fetch-deps (20200509.020918)
(2/5) Purging curl (7.67.0-r0)
(3/5) Purging libcurl (7.67.0-r0)
(4/5) Purging ca-certificates (20191127-r1)
Executing ca-certificates-20191127-r1.post-deinstall
(5/5) Purging nghttp2-libs (1.40.0-r0)
Executing busybox-1.31.1-r9.trigger
Executing glibc-bin-2.31-r0.trigger
/usr/glibc-compat/sbin/ldconfig: /usr/glibc-compat/lib/ld-linux-x86-64.so.2 is not a symbolic link

OK: 17 MiB in 17 packages
+ rm -rf '/var/cache/apk/*'
+ rm -rf /tmp/openjdk.tar.gz
Removing intermediate container 5a861f325886
 ---> 65d928ee55b8
Step 7/9 : ENV JAVA_HOME=/opt/java/openjdk     PATH="/opt/java/openjdk/bin:$PATH"
 ---> Running in 587330f6603b
Removing intermediate container 587330f6603b
 ---> 4c723c83f5d4
Step 8/9 : ENV JAVA_TOOL_OPTIONS="-XX:+IgnoreUnrecognizedVMOptions -XX:+UseContainerSupport -XX:+IdleTuningCompactOnIdle -XX:+IdleTuningGcOnIdle"
 ---> Running in 95957f1a2a21
Removing intermediate container 95957f1a2a21
 ---> 335adfbeeca0
Step 9/9 : CMD ["jshell"]
 ---> Running in ffa361fa1540
Removing intermediate container ffa361fa1540
 ---> cdd09d947ca9
Successfully built cdd09d947ca9
Successfully tagged openjdk:11.0-openj9
```

# 使用Jlink精简JDK

做了一下小服务，不需要乱七八糟的依赖，就需要`java.base和java.logging`就行了，所以上手jlink，干掉多余的东西。

## Jlink介绍

官方文档：[jlink](https://docs.oracle.com/en/java/javase/11/tools/jlink.html)

JEP：[JEP 282: jlink: The Java Linker](https://openjdk.java.net/jeps/282)

> 您可以使用jlink工具将一组模块及其依赖项组装和优化为自定义运行时映像。
> 
> ps. 这也就是OpenJDK9+没有再提供jre的原因，这个比jre强多了好吗，想咋改就咋改。

Jlink使用介绍：[Using jlink to Build Java Runtimes for non-Modular Applications](https://medium.com/azulsystems/using-jlink-to-build-java-runtimes-for-non-modular-applications-9568c5e70ef4)

**注：**如果想精简JDK镜像的话，先用`jdeps`查看下Jar包的依赖。做到心中有数。[jdeps link](https://docs.oracle.com/en/java/javase/11/tools/jdeps.html)

### 常用参数介绍

**--add-modules mod[,mod...]**

添加模块，后面加模块名的可变参数

**-c={0|1|2} or --compress={0|1|2}**

启用资源压缩

- 0：不压缩
- 1：常量字符串共享
- 2：ZIP

**--no-header-files**

排除头文件

**--no-man-pages**

排除用户手册

**--output** **path**

指定新JDK镜像位置

**--strip-debug**

不输出debug信息

## 参数示例

以我这边一个精简为例：

- 去掉头文件；
- 去掉用户手册；
- 不压缩（直接使用）；
- 屏蔽debug信息；
- 添加指定模块： `java.base,java.logging,jdk.unsupported`；
- 输出到指定目录：`/opt/openjdk/jre`

```
jlink --no-header-files --no-man-pages --compress=0 --strip-debug \
--add-modules java.base,java.logging,jdk.unsupported \
--output /opt/openjdk/jre;
```

基本上一个270Mb的HotSpot镜像，按照上面的需求精简完就40多Mb。

# 精简OpenJ9

用的命令和示例中相同，但是因为OpenJ9的JDK本来就大，所以精简完也会比HotSpot的大上10Mb左右。相较于内存节省的空间，在可接受范围内。

OpenJ9的精简基于前面构建的`openjdk:11.0-openj9`，通过Dockerfile的分层构建和基础镜像实现。具体思路如下：

1. 选择`openjdk:11.0-openj9`作为来源，精简其中的JDK，存放于指定目录。
2. 重新拉取一个Alpine镜像作为底层，拷贝刚才制作的JDK镜像到新的Alpine中。
3. 本地化配置

精简所用的Dockerfile详见附录

## 内存占用

通过OpenJ9镜像创建一个现有服务镜像，长时间稳定运行后，和之前对比。

测试用的是一个Spring Boot的MVC小项目，通过docker stats 查看内存占用。

说明：以`client-demo`镜像为例，长时间运行（24H以上）后对比。

```
docker stats <service's-docker-name>
```

Hotspot：

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/docker/hotspot-mem-use.png)

OpenJ9：

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/docker/openj9-mem-use.png)

可以明显的看到，Hotspot JVM的内存占用远高于OpenJ9。具体原因未分析，先用着

# 附：Docker JVM本地化

JVM的Docker版本都是官方原版的，所以有很多的配置都是通用配置，下面说下几个修改点。

## 时区

时区是有JVM所在的基础镜像所控制的，如果没有在jar运行时手动配置，则会直接调用系统时区。所以为了一劳永逸，直接修改基础镜像的时区，这里以Alpine为例：

Alpine配置时区需要安装`tzdata`，首先替换仓库为中科大的镜像，然后安装`tzdata`，配置完后再删除即可。因为是在一条命令中执行的，所以构建好后只会有一层。

```dockerfile
...

RUN set -eux; \
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories; \
    apk add --no-cache --virtual .build-deps binutils tzdata; \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
    echo "Asia/Shanghai" > /etc/timezone; \
    apk del tzdata .build-deps;

...
```

## 语言

系统默认语言基本都是US，需要修改为CN才可以。这个其实在上面 `构建JDK` 的时候在输出中出现过了。主要就是设置下Dockerfile的 `ENV` 语法糖。然后配置下自启动配置就好了。

```dockerfile
...
ENV LANG="zh_CN.UTF-8" LC_ALL="zh_CN.UTF-8"

RUN set -eux; \
    echo "export LANG=$LANG" > /etc/profile.d/locale.sh;

...
```

# 附：Dockerfile

## OpenJ9-full

- 本地编译适配，修改URL等
- 替换语言环境变量为中文

```dockerfile
FROM alpine:3.11
MAINTAINER Gsealy <gsealy@outlook.com>

ENV LANG="zh_CN.UTF-8" LC_ALL="zh_CN.UTF-8"

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk add --no-cache --virtual .build-deps curl binutils \
    && GLIBC_VER="2.31-r0" \
    && URL_PREFIX="http://10.20.61.27:8887" \
    && GCC_LIBS_URL="${URL_PREFIX}/gcc-libs-9.1.0-2-x86_64.pkg.tar.xz" \
    && ZLIB_URL="${URL_PREFIX}/zlib-1_1.2.11-3-x86_64.pkg.tar.xz" \
    && curl -LfsS ${URL_PREFIX}/sgerrand.rsa.pub -o /etc/apk/keys/sgerrand.rsa.pub \
    && curl -LfsS ${URL_PREFIX}/glibc-${GLIBC_VER}.apk > /tmp/glibc-${GLIBC_VER}.apk \
    && apk add --no-cache /tmp/glibc-${GLIBC_VER}.apk \
    && curl -LfsS ${URL_PREFIX}/glibc-bin-${GLIBC_VER}.apk > /tmp/glibc-bin-${GLIBC_VER}.apk \
    && apk add --no-cache /tmp/glibc-bin-${GLIBC_VER}.apk \
    && curl -Ls ${URL_PREFIX}/glibc-i18n-${GLIBC_VER}.apk > /tmp/glibc-i18n-${GLIBC_VER}.apk \
    && apk add --no-cache /tmp/glibc-i18n-${GLIBC_VER}.apk \
    && /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 "$LANG" || true \
    && echo "export LANG=$LANG" > /etc/profile.d/locale.sh \
    && curl -LfsS ${GCC_LIBS_URL} -o /tmp/gcc-libs.tar.xz \
    && mkdir /tmp/gcc \
    && tar -xf /tmp/gcc-libs.tar.xz -C /tmp/gcc \
    && mv /tmp/gcc/usr/lib/libgcc* /tmp/gcc/usr/lib/libstdc++* /usr/glibc-compat/lib \
    && strip /usr/glibc-compat/lib/libgcc_s.so.* /usr/glibc-compat/lib/libstdc++.so* \
    && curl -LfsS ${ZLIB_URL} -o /tmp/libz.tar.xz \
    && mkdir /tmp/libz \
    && tar -xf /tmp/libz.tar.xz -C /tmp/libz \
    && mv /tmp/libz/usr/lib/libz.so* /usr/glibc-compat/lib \
    && apk del --purge .build-deps glibc-i18n \
    && rm -rf /tmp/*.apk /tmp/gcc /tmp/gcc-libs.tar.xz /tmp/libz /tmp/libz.tar.xz /var/cache/apk/*

ENV JAVA_VERSION jdk-11.0.7+10_openj9-0.20.0

RUN set -eux; \
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories; \
    URL_PREFIX="http://10.20.61.27:8887"; \
    apk add --no-cache --virtual .fetch-deps curl; \
    curl -LfsSo /tmp/openjdk.tar.gz ${URL_PREFIX}/OpenJDK11U-jdk_x64_linux_openj9_11.0.7_10_openj9-0.20.0.tar.gz ; \
    mkdir -p /opt/java/openjdk; \
    cd /opt/java/openjdk; \
    tar -xf /tmp/openjdk.tar.gz --strip-components=1; \
    apk del --purge .fetch-deps; \
    rm -rf /var/cache/apk/*; \
    rm -rf /tmp/openjdk.tar.gz;

ENV JAVA_HOME=/opt/java/openjdk \
    PATH="/opt/java/openjdk/bin:$PATH"
ENV JAVA_TOOL_OPTIONS="-XX:+IgnoreUnrecognizedVMOptions -XX:+UseContainerSupport -XX:+IdleTuningCompactOnIdle -XX:+IdleTuningGcOnIdle"
CMD ["jshell"]
```

## Openj9-slim

- 包括full所有修改
- 使用`jlink`精简jdk
- 设定时区为`Asia/shanghai`

```dockerfile
FROM openjdk:11.0-openj9 as deps

RUN set -eux; \
    jlink --no-header-files --no-man-pages --compress=0 --strip-debug \
    --add-modules java.base,java.logging,jdk.unsupported \
    --output /opt/openjdk/jre;

FROM alpine:3.11
MAINTAINER Gsealy <gsealy@outlook.com>

ENV LANG="zh_CN.UTF-8" LC_ALL="zh_CN.UTF-8"

COPY --from=deps /opt/openjdk/jre /opt/openjdk/jre

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk add --no-cache --virtual .build-deps curl binutils \
    && GLIBC_VER="2.31-r0" \
    && URL_PREFIX="http://10.20.61.27:8887" \
    && GCC_LIBS_URL="${URL_PREFIX}/gcc-libs-9.1.0-2-x86_64.pkg.tar.xz" \
    && ZLIB_URL="${URL_PREFIX}/zlib-1_1.2.11-3-x86_64.pkg.tar.xz" \
    && curl -LfsS ${URL_PREFIX}/sgerrand.rsa.pub -o /etc/apk/keys/sgerrand.rsa.pub \
    && curl -LfsS ${URL_PREFIX}/glibc-${GLIBC_VER}.apk > /tmp/glibc-${GLIBC_VER}.apk \
    && apk add --no-cache /tmp/glibc-${GLIBC_VER}.apk \
    && curl -LfsS ${URL_PREFIX}/glibc-bin-${GLIBC_VER}.apk > /tmp/glibc-bin-${GLIBC_VER}.apk \
    && apk add --no-cache /tmp/glibc-bin-${GLIBC_VER}.apk \
    && curl -Ls ${URL_PREFIX}/glibc-i18n-${GLIBC_VER}.apk > /tmp/glibc-i18n-${GLIBC_VER}.apk \
    && apk add --no-cache /tmp/glibc-i18n-${GLIBC_VER}.apk \
    && /usr/glibc-compat/bin/localedef --force --inputfile POSIX --charmap UTF-8 "$LANG" || true \
    && echo "export LANG=$LANG" > /etc/profile.d/locale.sh \
    && curl -LfsS ${GCC_LIBS_URL} -o /tmp/gcc-libs.tar.xz \
    && mkdir /tmp/gcc \
    && tar -xf /tmp/gcc-libs.tar.xz -C /tmp/gcc \
    && mv /tmp/gcc/usr/lib/libgcc* /tmp/gcc/usr/lib/libstdc++* /usr/glibc-compat/lib \
    && strip /usr/glibc-compat/lib/libgcc_s.so.* /usr/glibc-compat/lib/libstdc++.so* \
    && curl -LfsS ${ZLIB_URL} -o /tmp/libz.tar.xz \
    && mkdir /tmp/libz \
    && tar -xf /tmp/libz.tar.xz -C /tmp/libz \
    && mv /tmp/libz/usr/lib/libz.so* /usr/glibc-compat/lib \
    && apk del --purge .build-deps glibc-i18n \
    && rm -rf /tmp/*.apk /tmp/gcc /tmp/gcc-libs.tar.xz /tmp/libz /tmp/libz.tar.xz /var/cache/apk/*

RUN set -eux; \
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories; \
    apk add --no-cache --virtual .build-deps binutils tzdata; \
    echo "export LANG=$LANG" > /etc/profile.d/locale.sh; \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
    echo "Asia/Shanghai" > /etc/timezone; \
    apk del tzdata .build-deps;

ENV JAVA_HOME=/opt/openjdk/jre \
    PATH=/opt/openjdk/jre/bin:$PATH
ENV JAVA_TOOL_OPTIONS="-XX:+IgnoreUnrecognizedVMOptions -XX:+UseContainerSupport -XX:+IdleTuningCompactOnIdle -XX:+IdleTuningGcOnIdle"
```

## Hotspot-slim

Alpine官方已经提供apk，直接精简就可以了

```dockerfile
FROM alpine:3.11 as deps

RUN set -eux; \
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories; \
    apk --no-cache add openjdk11-jdk openjdk11-jmods; \
    /usr/lib/jvm/java-11-openjdk/bin/jlink --no-header-files --no-man-pages --compress=0 --strip-debug \
    --add-modules java.base,java.logging,jdk.unsupported \
    --output /opt/openjdk/jre;

FROM alpine:3.11
MAINTAINER Gsealy <gsealy@outlook.com>

ENV LANG="zh_CN.UTF-8" LC_ALL="zh_CN.UTF-8"

COPY --from=deps /opt/openjdk/jre /opt/openjdk/jre

RUN set -eux; \
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories; \
    apk add --no-cache --virtual .build-deps binutils tzdata; \
    echo "export LANG=$LANG" > /etc/profile.d/locale.sh; \
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
    echo "Asia/Shanghai" > /etc/timezone; \
    apk del tzdata .build-deps;

ENV JAVA_HOME=/opt/openjdk/jre \
    PATH="/opt/openjdk/jre/lib:/opt/openjdk/jre/bin:$PATH"
```

总结：要是为了镜像大小的话，可以用OpenJDK官方的，因为支持musl-libc，所以会比OpenJ9的小几十MB。