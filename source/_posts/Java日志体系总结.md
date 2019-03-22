title: Java日志体系总结
author: alben.wong
abbrlink: 854fc091
tags:
  - log
  - java日志
categories:
  - java
  - log
date: 2019-03-22 11:12:00
keywords: Java日志体系 Java-Log
description: 本文的目的是搞清楚Java中各种日志Log之间是怎么的关系，如何作用、依赖，好让我们平时在工作中如果遇到“日志打不出”或者“日志jar包冲突”等之类的问题知道该如何入手解决，以及在各种场景下如何调整项目中的各个框架的日志输出，使得输出统一。
---
## 概要
本文的目的是搞清楚Java中各种日志Log之间是怎么的关系，如何作用、依赖，好让我们平时在工作中如果遇到“日志打不出”或者“日志jar包冲突”等之类的问题知道该如何入手解决，以及在各种场景下如何调整项目中的各个框架的日志输出，使得输出统一。

## Log日志体系
在日常工作中我们可能看到项目中依赖的跟日志相关的jar包有很多，`commons-logging.jar`、`log4j.jar`、`sl4j-api.jar`、`logback.jar`等等，眼花缭乱。我们要正确的配置，使得jar包相互作用生效之前，就先要理清它们之间的关系。

### 背景/发展史
那就要从Java Log的发展历程开始说起。
1. `log4j`（作者Ceki Gülcü）出来时就等到了广泛的应用（注意这里是直接使用），是Java日志事实上的标准，并成为了Apache的项目
2. Apache要求把log4j并入到JDK，SUN拒绝，并在jdk1.4版本后增加了`JUL`（`java.util.logging`）
3. 毕竟是JDK自带的，JUL也有很多人用。同时还有其他日志组件，如SimpleLog等。这时如果有人想换成其他日志组件，如log4j换成JUL，因为api完全不同，就需要改动代码。
4. Apache见此，开发了`JCL`（Jakarta Commons Logging），即`commons-logging-xx.jar`。它只提供一套通用的日志接口api，并不提供日志的实现。很好的设计原则嘛，依赖抽象而非实现。这样应用程序可以在运行时选择自己想要的日志实现组件。
5. 这样看上去也挺美好的，但是log4j的作者觉得JCL不好用，自己开发出`slf4j`，它跟JCL类似，本身不替供日志具体实现，只对外提供接口或门面。目的就是为了替代JCL。同时，还开发出`logback`，一个比log4j拥有更高性能的组件，目的是为了替代log4j。
6. Apache参考了logback,并做了一系列优化，推出了`log4j2`

### 关系/依赖
大概了解心路历程后，再详细看看它们之间的关系、依赖。

#### JCL
`commons-logging`已经停止更新，最后的状态如下所示：
![upload successful](/images/Java日志体系总结__0.png)
JCL支持日志组件不多，不过也有很人用的，例如Spring
现在用的也越来越少了，也不多讲了

#### SLF4J
因为当时Java的日志组件比较混乱繁杂，Ceki Gülcü推出slf4j后，也相应为行业中各个主流日志组件推出了slf4j的适配
图来源于[官方文档](https://www.slf4j.org/manual.html)

![upload successful](/images/java日志体系总结__2.png)

图的意思为如果你想用slf4j作为日志门面的话，你如何去配合使用其他日志实现组件，这里说明一下（注意jar包名缺少了版本号，在找版本时也要注意版本之间是否兼容）
- slf4j + logback
`slf4j-api.jar` + `logback-classic.jar` + `logback-core.jar`
- slf4j + log4j
`slf4j-api.jar` + `slf4j-log4j12.jar` + `log4j.jar`
- slf4j + jul
`slf4j-api.jar` + `slf4j-jdk14.jar`
- 也可以只用slf4j无日志实现
`slf4j-api.jar` + `slf4j-nop.jar`

### SLF4J的适配
slf4j支持各种适配，无论你现在是用哪种日志组件，你都可以通过slf4j的适配器来使用上slf4j。
只要你切换到了slf4j，那么再通过slf4j用上实现组件，即上面说的。
图来源于[官方文档](https://www.slf4j.org/legacy.html)

![upload successful](/images/Java日志体系总结__1.png)

其实总的来说，无论就是以下几种情况
- 你在用JCL
使用`jcl-over-slf4j.jar`适配
- 你在用log4j
使用`log4j-over-slf4j.jar`适配
- 你在用JUL
使用`jul-to-slf4j.jar`适配

我在网上盗一张图，给大家一个整体的依赖图（懒得画了）

![upload successful](/images/Java日志体系总结__3.png)

#### 让Spring统一输出
这就是为了对slf4j的适配做一个例子说明。
Spring是用JCL作为日志门面的，那我们的应用是slf4j + logback，怎么让Spring也用到logback作为日志输出呢？这样的好处就是我们可以统一项目内的其他模块、框架的日志输出（日志格式，日志文件，存放路径等，以及其他slf4j支持的功能）
很简单，就是加入`jcl-over-slf4j.jar`就好了。
我又盗了一个图来说明

![upload successful](/images/Java日志体系总结__4.png)

#### 适配思路
其实很简单
1. 你首先确认需要统一日志的模块、框架是使用哪个日志组件的，然后再找到sfl4j的适配器。
2. 记得去掉无用的日志实现组件，只保留你要用的。

## 常见问题
slf4j的日志加载会在程序启动时把日志打出来，所以一定要注意，它会说明加载是否成功，加载了那个日志实现。
slf4j已经对错误作了说明[官网说明](https://www.slf4j.org/codes.html)
下面讲一下可能经常遇到的问题

### Failed to load class org.slf4j.impl.StaticLoggerBinder
没找到日志实现，如果你觉得你已经写上了对应的日志实现依赖了，那你要检查一下了，一般来说极有可能是版本不兼容。

### Multiple bindings
找到多个日志实现，slf4j会找其中一个作为日志实现。

## 总结
文章帮大家梳理了Java日志组件的关系，以及如何解决日常中常见日志相关的问题，希望对大家帮助。

## 参考资料
[架构师必备，带你弄清混乱的JAVA日志体系！](https://www.javazhiyin.com/27585.html)
[10分钟搞定--混乱的 Java 日志体系](https://www.jianshu.com/p/39ced06944a2)
[Java主流日志工具介绍和使用](http://hbyou.me/2016/06/18/Java%E4%B8%BB%E6%B5%81%E6%97%A5%E5%BF%97%E5%B7%A5%E5%85%B7%E4%BB%8B%E7%BB%8D%E5%92%8C%E4%BD%BF%E7%94%A8/)
https://www.slf4j.org/