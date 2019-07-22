---
title: IDEA不能更新Spring快照版本
tags:
  - Maven
  - IDEA
abbrlink: 4980ebc2
date: 2019-02-19 13:53:10
---

> IDEA 版本：IntelliJ IDEA 2018.3.4 (Ultimate Edition)
>
> Maven版本：3.5.2
>
> Java版本：1.8

注：应该是所有版本都可以试用，但是IDEA的设置页面不保证完全不变。

从`Git`上克隆了 `Spring Cloud Gateway`，但是在IDEA里面构建项目的时候犯了难。因为都是快照版本的，拉取不下来。网上找了一圈，都是让勾选下图的`Always update snapshots`就行了。

![](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/spring/IDEA-Setting.png)

但是我勾选之后，再`Reimport`就好了。但是对于我来说，一点用都没有。

重新查资料以后，我发现，因为没有拉取`maven-metadata.xml`到本地，所以无法下载jar，使用命令强制更新就好，但是会出Error，不用管它。再去IDEA重新`Reimport`就可以拉取jar了

```
> mvn -U
[INFO] Scanning for projects...
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-build/2.0.5.BUILD-SNAPSHOT/maven-met
adata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-build/2.0.5.BUILD-SNAPSHOT/maven-meta
data.xml (636 B at 740 B/s)
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-dependencies-parent/2.0.5.BUILD-SNAP
SHOT/maven-metadata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-dependencies-parent/2.0.5.BUILD-SNAPS
HOT/maven-metadata.xml (650 B at 1.3 kB/s)
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-commons-dependencies/2.0.1.BUILD-SNA
PSHOT/maven-metadata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-commons-dependencies/2.0.1.BUILD-SNAP
SHOT/maven-metadata.xml (651 B at 1.3 kB/s)
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-dependencies-parent/2.0.2.BUILD-SNAP
SHOT/maven-metadata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-dependencies-parent/2.0.2.BUILD-SNAPS
HOT/maven-metadata.xml (650 B at 1.3 kB/s)
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-netflix-dependencies/2.0.1.BUILD-SNA
PSHOT/maven-metadata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-netflix-dependencies/2.0.1.BUILD-SNAP
SHOT/maven-metadata.xml (651 B at 1.3 kB/s)
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-dependencies-parent/2.0.3.BUILD-SNAP
SHOT/maven-metadata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-dependencies-parent/2.0.3.BUILD-SNAPS
HOT/maven-metadata.xml (650 B at 1.3 kB/s)
Downloading from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-build-dependencies/2.0.5.BUILD-SNAPS
HOT/maven-metadata.xml
Downloaded from spring-snapshots: https://repo.spring.io/libs-snapshot-local/org/springframework/cloud/spring-cloud-build-dependencies/2.0.5.BUILD-SNAPSH
OT/maven-metadata.xml (649 B at 833 B/s)
[WARNING]
[WARNING] Some problems were encountered while building the effective model for org.springframework.cloud:spring-cloud-gateway-core:jar:2.0.3.BUILD-SNAPS
HOT
[WARNING] 'build.plugins.plugin.version' for org.apache.maven.plugins:maven-compiler-plugin is missing. @ org.springframework.cloud:spring-cloud-gateway-
core:[unknown-version], D:\Git\spring-cloud-gateway\spring-cloud-gateway-core\pom.xml, line 157, column 12
[WARNING] 'build.plugins.plugin.version' for org.jetbrains.kotlin:kotlin-maven-plugin is missing. @ org.springframework.cloud:spring-cloud-gateway-core:[
unknown-version], D:\Git\spring-cloud-gateway\spring-cloud-gateway-core\pom.xml, line 127, column 12
[WARNING]
[WARNING] Some problems were encountered while building the effective model for org.springframework.cloud:spring-cloud-gateway-sample:jar:2.0.3.BUILD-SNA
PSHOT
[WARNING] 'build.plugins.plugin.version' for org.apache.maven.plugins:maven-compiler-plugin is missing. @ org.springframework.cloud:spring-cloud-gateway-
sample:[unknown-version], D:\Git\spring-cloud-gateway\spring-cloud-gateway-sample\pom.xml, line 127, column 12
[WARNING] 'build.plugins.plugin.version' for org.jetbrains.kotlin:kotlin-maven-plugin is missing. @ org.springframework.cloud:spring-cloud-gateway-sample
:[unknown-version], D:\Git\spring-cloud-gateway\spring-cloud-gateway-sample\pom.xml, line 89, column 12
[WARNING] 'build.plugins.plugin.version' for org.apache.maven.plugins:maven-deploy-plugin is missing. @ org.springframework.cloud:spring-cloud-gateway-sa
mple:[unknown-version], D:\Git\spring-cloud-gateway\spring-cloud-gateway-sample\pom.xml, line 162, column 12
[WARNING]
[WARNING] Some problems were encountered while building the effective model for org.springframework.cloud:spring-cloud-gateway-docs:pom:2.0.3.BUILD-SNAPS
HOT
[WARNING] 'build.plugins.plugin.version' for org.apache.maven.plugins:maven-deploy-plugin is missing. @ org.springframework.cloud:spring-cloud-gateway-do
cs:[unknown-version], D:\Git\spring-cloud-gateway\docs\pom.xml, line 21, column 12
[WARNING]
[WARNING] It is highly recommended to fix these problems because they threaten the stability of your build.
[WARNING]
[WARNING] For this reason, future Maven versions might no longer support building such malformed projects.
[WARNING]
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Build Order:
[INFO]
[INFO] spring-cloud-gateway-dependencies
[INFO] Spring Cloud Gateway
[INFO] spring-cloud-gateway-mvc
[INFO] spring-cloud-gateway-webflux
[INFO] Spring Cloud Gateway Core
[INFO] spring-cloud-starter-gateway
[INFO] Spring Cloud Gateway Sample
[INFO] Spring Cloud Gateway Docs
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO]
[INFO] spring-cloud-gateway-dependencies .................. SKIPPED
[INFO] Spring Cloud Gateway ............................... SKIPPED
[INFO] spring-cloud-gateway-mvc ........................... SKIPPED
[INFO] spring-cloud-gateway-webflux ....................... SKIPPED
[INFO] Spring Cloud Gateway Core .......................... SKIPPED
[INFO] spring-cloud-starter-gateway ....................... SKIPPED
[INFO] Spring Cloud Gateway Sample ........................ SKIPPED
[INFO] Spring Cloud Gateway Docs .......................... SKIPPED
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 10.046 s
[INFO] Finished at: 2019-02-19T13:50:51+08:00
[INFO] Final Memory: 17M/160M
[INFO] ------------------------------------------------------------------------
[ERROR] No goals have been specified for this build. You must specify a valid lifecycle phase or a goal in the format <plugin-prefix>:<goal> or <plugin-g
roup-id>:<plugin-artifact-id>[:<plugin-version>]:<goal>. Available lifecycle phases are: validate, initialize, generate-sources, process-sources, generat
e-resources, process-resources, compile, process-classes, generate-test-sources, process-test-sources, generate-test-resources, process-test-resources, t
est-compile, process-test-classes, test, prepare-package, package, pre-integration-test, integration-test, post-integration-test, verify, install, deploy
, pre-clean, clean, post-clean, pre-site, site, post-site, site-deploy. -> [Help 1]
[ERROR]
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR]
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/NoGoalSpecifiedException
```

结束！🔚

------

