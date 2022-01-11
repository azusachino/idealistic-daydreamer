---
title: Understand Java Logger (hopefully)
description: 世上本没有那么多技术，场景多了，技术也多了
date: 2022-01-11
slug: understand-java-logger
image: img/2022/01/SnowBuntings.jpg
categories:
  - conclusion
tags:
  - java
---

在处理客户端日志文件的过程中，出现了在 ElasticSearch 中的数据在同一毫秒级的情况下无法区分数据的问题。通过在网络上查询资料，最终采用秒级时间戳 + 文件内 offset 的方式进行联合排序。(其中 offset 由 filebeat 统一生成)

## 业务逻辑调整

### Old

通过 http 上传日志文件 -> http-nio 线程异步读取文件内容 -> 解压保存至指定目录 -> 异步线程按行读取 -> 逐一发送至 kafka。

### New

通过 http 上传日志文件 -> http-nio 线程异步读取文件内容 -> 在内存中解压，按行读取 -> 通过 Logger 输出到指定路径 -> filebeat 采集输出到 redis。

## Logger 实现原理

Logger 相关的两个主要方法的源码追踪如下，其中 `LoggerContext` 是 `LoggerFactory` 的实现类。

```java
// Logger Creation
LoggerFactory.getLogger(String)
├── LoggerFactory.getILoggerFactory() // 获取Logger工厂类
│   │── LoggerFactory.performInitialization() // 初始化LoggerFactory
│   │   └── LoggerFactory.bind() // 绑定
│   │       └── StaticLoggerBinder.getSingleton() // 获取绑定类单例
│   │           └── StaticLoggerBinder.init()
│   └── StaticLoggerBinder.getSingleton().getLoggerFactory() // 获取最终的LoggerFactory
└── LoggerContext.getLogger(String) // 创建或获取Logger

// Real log creation (with SLF4J & Logback), in case of RollingFileAppender
Logger.trace(Marker, String, Object arg) // 打印trace日志
├── Logger.filterAndLog_1(String, Marker, Level, String, Object, Throwable)
│   │── LoggerContext.getTurboFilterChainDecision_1() // 判断当前level是否可以输出
│   └── Logger.buildLoggingEventAndAppend(String, Marker, Level, String, Object[], Throwable) // 创建ILoggingEvent并输出到Appender
│          └── Logger.callAppenders(ILoggingEvent) // Invoke all the appenders of this logger.
│               └── Logger.appendLoopOnAppenders(ILoggingEvent)
│                   └── AppenderAttachableImpl.appendLoopOnAppenders(ILoggingEvent) // 将事件追加到数组中
│                       └── Appender.doAppend(E) // 调用每个Appender处理Event
│                           └── UnsynchronizedAppenderBase.append(E)
│                               └── OutputStreamAppender.subAppend(E)
│                                   │── Encoder.encode(E) // 将Event转译成字节数组
│                                   └── OutputStreamAppender.writeBytes(byte[])
│                                       └── OutputStream.write(byte[]) // 实际写入到OutStream中，此次是RollingFile

RollingFileAppender主要负责处理文件和滚动逻辑。
```

### LoggerContext

ROOT 是所有 Logger 的 parent，初始化时就会生成好。

```java
public class LoggerContext extends ContextBase implements ILoggerFactory, LifeCycle {
  // ROOT Logger, initialized at construction
  final Logger root;
  // local logger storage
  private Map<String, Logger> loggerCache;
}
```

### Logger

```java
public final class Logger implements org.slf4j.Logger, LocationAwareLogger, AppenderAttachable<ILoggingEvent>, Serializable {
  // current logger name
  private String name;
  // logging level
  private Level level;
  // logger parent pointer
  private Logger parent;
  // logger children list
  private List<Logger> childrenList;
  // Combined Appender list
  private AppenderAttachableImpl<ILoggingEvent> aai;
  // Additivity is set to true by default, that is children inherit the appenders of their ancestors by default.
  private boolean active = true;
  // logger Context (aka LoggerFactory)
  private LoggerContext loggerContext;
}
```

## 参考文件 logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="true" scan="true" scanPeriod="30 minutes">

    <springProperty scope="context" name="logDir" source="logging.file.path" defaultValue="/usr/local/app/logs"/>
    <springProperty scope="context" name="appName" source="spring.application.name"/>

    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>

    <property name="CONSOLE_LOG_PATTERN"
              value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

    <!-- 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>info</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="logstashAppender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${logDir}/log.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${logDir}/log.log.%i.%d{yyyy-MM-dd}.gz</fileNamePattern>
            <maxHistory>3</maxHistory>
            <maxFileSize>100MB</maxFileSize>
        </rollingPolicy>
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder"><!-- 必须指定，否则不会往文件输出内容 -->
            <providers>
                <pattern>
                    <pattern>
                        {
                        "t":"%d{yyyy-MM-dd HH:mm:ss.SSS,Asia/Shanghai}",
                        "app":"${appName}",
                        "level":"%level",
                        "pid":"${PID:- }",
                        "thread":"%15.15t",
                        "logger":"%-40.40logger{39}",
                        "line":"%line",
                        "stack_trace": "%exception{255}",
                        "method":"%method",
                        "msg":"%msg"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
        <append>false</append>
        <prudent>false</prudent>
    </appender>

    <!-- bind logger with appender -->
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="logstashAppender"/>
    </root>
</configuration>
```

## 最后

我差不多明白了，你呢。如果有不懂的地方，欢迎一起沟通交流。
