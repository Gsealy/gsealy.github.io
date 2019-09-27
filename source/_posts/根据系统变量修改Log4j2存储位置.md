---
title: 根据系统变量修改Log4j2存储位置
date: 2019-09-27 09:31:41
tags:
- Java
- Log4j2
---

# 前言

在某些特定环境下，需要指定`log`的存储位置，所以在启动前需要动态的更改`log.home`这一自定义参数， 例：生产环境直接部署jar包，因为`log4j2.xml`配置文件设置了默认的log存储，需要通过系统参数修改文件位置。位置经查，Log4j是支持部分自定义参数的。

# 查看官方文档

在`Log4j2`的[Property Substitution](http://logging.apache.org/log4j/2.x/manual/configuration.html#PropertySubstitution)一节，有相关描述

> Log4j 2 supports the ability to specify tokens in the configuration as references to properties defined elsewhere. Some of these properties will be resolved when the configuration file is interpreted while others may be passed to components where they will be evaluated at runtime. To accomplish this, Log4j uses variations of [Apache Commons Lang](https://commons.apache.org/proper/commons-lang/)'s [StrSubstitutor](http://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrSubstitutor.html) and [StrLookup](http://logging.apache.org/log4j/2.x/log4j-core/apidocs/org/apache/logging/log4j/core/lookup/StrLookup.html) classes. In a manner similar to Ant or Maven, this allows variables declared as `${name}` to be resolved using properties declared in the configuration itself. For example, the following example shows the filename for the rolling file appender being declared as a property.

基本意思就是可以通过指定的引用标记做属性替换，具体属性如下：

在表中，`sys`参数能够解决当前问题，按照示例的格式修改配置文件，指定默认的位置是`logs`文件夹，同时也支持系统参数指定

```xml
<Property name="LOG_HOME">${sys:log.home:-logs}</Property>
```

然后在jar启动的时候通过`-D`传入即可， 例：

```
> java -jar xxx-fat.jar -Dlog.home=newlogs
```

到同级目录就会看到`newlogs`目录和其中的log文件

# 完整模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="error">  <Properties>    
<?xml version="1.0" encoding="utf-8"?>

<configuration status="error"> 
  <Properties> 
    <Property name="LOG_HOME">${sys:log.home:-logs}</Property>  
    <property name="ERROR_LOG_FILE_NAME">${LOG_HOME}/error</property>  
    <property name="WARN_LOG_FILE_NAME">${LOG_HOME}/warn</property>  
    <property name="INFO_LOG_FILE_NAME">${LOG_HOME}/info</property>  
    <property name="DEBUG_LOG_FILE_NAME">${LOG_HOME}/debug</property>  
    <property name="PATTERN">[%d{yyyy-MM-dd HH:mm:ss}] [%t] %-5p [%c] %L - %m%n</property> 
  </Properties>  
  <appenders> 
    <Console name="Console" target="SYSTEM_OUT"> 
      <!--只接受程序中DEBUG级别的日志进行处理, 下同 -->  
      <ThresholdFilter level="DEBUG" onMatch="ACCEPT" onMismatch="DENY"/>  
      <PatternLayout pattern="${PATTERN}"/> 
    </Console>  
    <RollingFile fileName="${DEBUG_LOG_FILE_NAME}.log" filePattern="logs/$${date:yyyy-MM}/debug-%d{yyyy-MM-dd}-%i.log.gz" name="RollingFileDebug"> 
      <Filters> 
        <ThresholdFilter level="DEBUG"/>  
        <ThresholdFilter level="INFO" onMatch="DENY" onMismatch="NEUTRAL"/> 
      </Filters>  
      <PatternLayout pattern="${PATTERN}"/>  
      <Policies> 
        <SizeBasedTriggeringPolicy size="50 MB"/>  
        <TimeBasedTriggeringPolicy/> 
      </Policies>  
      <!-- max:同一文件夹下最多文件数 -->  
      <DefaultRolloverStrategy max="10"> 
        <Delete basePath="${LOG_HOME}" maxDepth="2"> 
          <IfFileName glob="*/debug*"> 
            <!-- 保存天数 -->  
            <IfLastModified age="10d"> 
              <IfAny> 
                <IfAccumulatedFileSize exceeds="100 MB"/>  
                <IfAccumulatedFileCount exceeds="10"/> 
              </IfAny> 
            </IfLastModified> 
          </IfFileName> 
        </Delete> 
      </DefaultRolloverStrategy> 
    </RollingFile>  
    <RollingFile fileName="${INFO_LOG_FILE_NAME}.log" filePattern="logs/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.gz" name="RollingFileInfo"> 
      <Filters> 
        <ThresholdFilter level="INFO"/>  
        <ThresholdFilter level="WARN" onMatch="DENY" onMismatch="NEUTRAL"/> 
      </Filters>  
      <PatternLayout pattern="${PATTERN}"/>  
      <Policies> 
        <SizeBasedTriggeringPolicy size="50 MB"/>  
        <TimeBasedTriggeringPolicy/> 
      </Policies>  
    </RollingFile>  
    <RollingFile fileName="${WARN_LOG_FILE_NAME}.log" filePattern="logs/$${date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.gz" name="RollingFileWarn"> 
      <Filters> 
        <ThresholdFilter level="WARN"/>  
        <ThresholdFilter level="ERROR" onMatch="DENY" onMismatch="NEUTRAL"/> 
      </Filters>  
      <PatternLayout pattern="${PATTERN}"/>  
      <Policies> 
        <SizeBasedTriggeringPolicy size="50 MB"/>  
        <TimeBasedTriggeringPolicy/> 
      </Policies>  
    </RollingFile>  
    <RollingFile fileName="${ERROR_LOG_FILE_NAME}.log" filePattern="logs/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz" name="RollingFileError"> 
      <ThresholdFilter level="ERROR"/>  
      <PatternLayout pattern="${PATTERN}"/>  
      <Policies> 
        <SizeBasedTriggeringPolicy size="50 MB"/>  
        <TimeBasedTriggeringPolicy/> 
      </Policies>  
    </RollingFile> 
  </appenders>  
  <loggers> 
    <root level="debug" includeLocation="true"> 
      <appender-ref ref="Console"/>  
      <appender-ref ref="RollingFileInfo"/>  
      <appender-ref ref="RollingFileWarn"/>  
      <appender-ref ref="RollingFileError"/>  
      <appender-ref ref="RollingFileDebug"/> 
    </root> 
  </loggers>
</configuration>

```

结束！🔚

------

