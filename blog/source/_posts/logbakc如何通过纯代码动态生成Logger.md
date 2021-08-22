---
title: logbakc如何通过纯代码动态生成Logger
date: 2020-03-27 21:50:56
tags:  logback
---

有时候，我们想单独开发一个专门写日志的进程，用于收集一些日志。但是这些日志可能各种各样，要支持根据不同请求，动态写入不同目录和文件。在此情况下，传统的XML配置方式就行不通了，即使通过MDC实现，也只能实现有限的动态。那么如何通过纯代码来生成一个logbakc的Logger呢？
<!--more-->

## 一、基础知识
我们以一个最基本的日志xml配置片段来讲
```xml
<included>
    <appender name="sample_appender" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/data/%d{yyyyMMdd}.txt</fileNamePattern>
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder>
            <charset>UTF-8</charset>
            <pattern>%msg%n</pattern>
        </encoder>
    </appender>
    <appender name="sample_appender_async" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="sample_appender" />
    </appender>
    <logger name="sample_logger" additivity="false">
        <level>INFO</level>
        <appender-ref ref="sample_appender_async"/>
    </logger>

</included>
```

### 1、Appender
当我们需要写入日志时，会产生一个事件，这个事件会委托给Appender执行。例如上面配置文件中的sample_appender和sample_appender_async。
你可以看到，异步的那个Appender自己只是引用了一个前面的RollingAppender，是因为调用这个异步Appender的时候，会异步的委托给自己的引用的Appender，从而完成丰富的功能。

appender有很多种，例如自动滚动文件的Appender和异步Appender。在一个Appender，可以配置具体的滚动策略，例如上面的基于时间滚动的TimeBasedRollingPolicy

### 2、Policy
看上面的配置文件，用的是TimeBasedRollingPolicy表示按时间滚动。那么滚动的时候总要指定文件的命名，保留时长。通过fileNamePattern指定了文件名的格式，通过maxHistory指定了保留文件的最长时间

### 3、Pattern
policy中的pattern用来动态指定文件名格式的，例如/data/%d{yyyyMMdd}.txt表示按天滚动，文件名格式是 /data/yyyyMMdd.txt, 如果按小时滚动也可以，后面加上小时的精度就可以，例如/data/%{yyyyMMddHH}.txt

### 4、Encoder
encoder 不但可以完全控制字节写出时的格式，而且还可以控制这些字节什么时候被写出，例如我们配置了以UTF-8编码日志，%msg%n表示仅输出日志内容和与平台对应的换行符，更多变量请看 [变量参考](http://logback.qos.ch/manual/layouts.html)

### 5 Logger
Logger是我们用来执行函数的实例。Logger调用写日志的函数的时候，就会最终委托给Appender去执行写入操作。
Logger是有层级的，可以继承的。如果Logger的名字叫”abc”，另一个Logger的名字叫“abc.123”，因为.的存在，所以前面那个Logger就自然成了后者的父级。
ROOT Logger是层次的最高层，它是一个特殊的 logger，因为它是每一个层次结构的一部分。每一个 logger 都可以通过它的名字去获取。例：Logger rootLogger = LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME)
如果一个Logger叫L，当调用L的时候，那么日志会调用L和祖先的Appender。所以有时候你会发现，为啥一条日志出现在了好几个地方，是因为你没配置，但是他的父Logger配置了某个Appender。如果其中一个祖先Logger叫做P，P又设置了additivity = false,那么就上溯到 P（包含）为止。就像我们上面的配置例子，为了防止这个Logger调用时，又去调用祖先的Appender，所以设置了additivity="false"

整体结构就是
Logger —– Logger — Loggger {Appender { Policy, Encoder}}


## 动态生成Logger
有了上面的知识，那么生成Logger的就比较容易了，注意，每个组件需要调用Start()才可以起作用，如下：
```java
private Logger buildLogger(String fileDir, RotateType rotateType) {

       String loggerName = "一个你觉得不错的名字";
	// 获得一个Context，基本上整个进程就用这一个就行
       LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
	// 构造rollingAppender
       RollingFileAppender<ILoggingEvent> rollingFileAppender = new RollingFileAppender<ILoggingEvent>();
       // 记得要设置Context
       rollingFileAppender.setContext(loggerContext);
       rollingFileAppender.setAppend(true);
       // 这个名字随意啦，按你们项目里的规则起名
       rollingFileAppender.setName(loggerName + "_rolling_appender");
       
	// 给RollingAppender创建一个滚动规则
       TimeBasedRollingPolicy rollingPolicy = new TimeBasedRollingPolicy<>();
       rollingPolicy.setFileNamePattern("/data/%{yyyyMMdd}.txt");
       // 最长保存10天
       rollingPolicy.setMaxHistory(10);
       rollingPolicy.setContext(loggerContext);
       // 和RollingAppender挂一块
       rollingPolicy.setParent(rollingFileAppender);
       rollingPolicy.start();
	
	// 告诉rollingAppender，他有了一个滚动策略
       rollingFileAppender.setRollingPolicy(rollingPolicy);

	// 设置输出规则和编码方式
       PatternLayoutEncoder encoder = new PatternLayoutEncoder();
       encoder.setPattern("%msg%n");
       encoder.setCharset(Charset.forName("UTF-8"));
       encoder.setContext(loggerContext);
       encoder.start();
       rollingFileAppender.setEncoder(encoder);
       rollingFileAppender.start();

	// 如果需要异步，就创建一个，引用到RollingAppender就可以了
       AsyncAppender asyncAppender = new AsyncAppender();
       asyncAppender.setContext(loggerContext);
       asyncAppender.setName("一个不错的名字");
       asyncAppender.addAppender(rollingFileAppender);
       asyncAppender.start();
	// 最终返回的Logger
       ch.qos.logback.classic.Logger customLogger = loggerContext.getLogger(loggerName);
       // 如果不想受祖先Logger里Appender的干扰，设置成false
       customLogger.setAdditive(false);
       customLogger.setLevel(Level.INFO);
       customLogger.addAppender(asyncAppender);
       log.info("build logger finish, fileDir = {}, rotateType = {}", fileDir, rotateType);
       return customLogger;
}
```
运行时的Logger结构如下图，其中aai就是appender，可以看到这个Logger的Parent是名为””的Logger，因为以点号开头，他就是为点号前面的那一串也是一个，就偷偷构造了个父Logger。
而且我们创建的这个Logger祖先Logger都是有Appender的，如果我们不设置additive=fasle，那么也会执行祖先Logger的Appender。

![img](https://res.cloudinary.com/saintasmita/image/upload/v1585307802/codexiongan/%E4%B8%80%E4%B8%AALogger%E7%BB%93%E6%9E%84_yaz9m7.png)