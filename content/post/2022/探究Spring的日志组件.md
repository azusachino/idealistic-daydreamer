---
title: 探究 Spring 中 Logger 的实现
description: 代理模式的范式
date: 2022-01-11
slug: understand-java-logger
image: img/2022/01/SnowBuntings.jpg
mermaid: true
categories: [Conclusion]
tags: [Conclusion, Java]
---

由于工作中遇到了需要自定义日志打印的场景，所以针对 Spring 中常用的 slf4j 日志库进行了大致的学习；以下即是学习成果。

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

## Logback 配置文件详解

{{<mermaid>}}
flowchart LR
configuration --- appender
configuration --- logger
configuration --- root
{{</mermaid>}}

### configuration

### appender

encoder 负责两件事情：把日志信息转换成字节数组，把字节数组写入到输出流

### logger

### root

ROOT 是所有 Logger 的父节点，

## 参考文件 logback-spring.xml

### 普通 PatternLayoutEncoder

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">

    <!--定义日志文件的存储地址 勿在 LogBack 的配置中使用相对路径-->
    <property name="LOG_HOME" value="logs" />
    <property name="APP_NAME" defaultValue="spring.application.name"/>

    <!--控制台日志， 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度,%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <!--文件日志， 按照每天生成日志文件 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名-->
            <FileNamePattern>${LOG_HOME}/TestWeb.log.%d{yyyy-MM-dd}.log</FileNamePattern>
            <!--日志文件保留天数-->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
        <!--日志文件最大的大小-->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>10MB</MaxFileSize>
        </triggeringPolicy>
    </appender>

    <!-- show parameters for hibernate sql 专为 Hibernate 定制 -->
    <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE" />
    <logger name="org.hibernate.type.descriptor.sql.BasicExtractor" level="DEBUG" />
    <logger name="org.hibernate.SQL" level="DEBUG" />
    <logger name="org.hibernate.engine.QueryParameters" level="DEBUG" />
    <logger name="org.hibernate.engine.query.HQLQueryPlan" level="DEBUG" />

    <!--myibatis log configure-->
    <logger name="com.apache.ibatis" level="TRACE" />
    <logger name="java.sql.Connection" level="DEBUG" />
    <logger name="java.sql.Statement" level="DEBUG" />
    <logger name="java.sql.PreparedStatement" level="DEBUG" />

    <!-- 日志输出级别 -->
    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

### LoggingEventCompositeJsonEncoder

- appName
- pid
- line (消耗性能)
- stack trace (跨行输出，需要搭配 filebeat + grok)
- method

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
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
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

## References

- [logback 配置文件---logback.xml 详解](https://www.cnblogs.com/gavincoder/p/10091757.html)
- [logback pattern 配置及详解](https://blog.csdn.net/snail_bi/article/details/103496697)
