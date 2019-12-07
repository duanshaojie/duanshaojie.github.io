---
title: 从log4j转logback踩的坑
tags: log4j logback
categories: java
---


# 从log4j转logback踩的坑

​	因为一个未知的bug导致的需要去查找error日志，才发现这个项目不知从何时起，就已经不在打印log了，祖传项目果然厉害，然后开始了一段排查之旅:tw-203c: :tw-203c: :tw-203c:

<!--more-->
1. 解决无法打印日志的问题
------

​	这个问题比较好解决，在项目启动的时候发现有jar包冲突了，导致log4j日志没法打印。 因为我使用的是IDEA，所以这里也是使用IDEA做记录。

​	怎么排除冲突jar包可以看这篇文章[Intellij IDEA 中如何查看maven项目中所有jar包的依赖关系图](https://blog.csdn.net/qq_27093465/article/details/69226949 "Intellij IDEA 中如何查看maven项目中所有jar包的依赖关系图")

​	当我兴致冲冲准备提交上线的时候，路哥和我说这怎么还在用log4j呀，其他项目都是logback了。

2. log4j转logback
------

​	本来以为很简单，不就是删掉log4j.properties，加上logback.xml 几分钟搞点，测试成功~

​	但貌似不对呀，启动以后不停的刷日志，如下
```shell
DEBUG(ClientCnxn.java:758)ClientCnxn:758 - Got ping response for sessionid: 0x1018d58af04377b after 1ms
DEBUG(ClientCnxn.java:758)ClientCnxn:758 - Got ping response for sessionid: 0x1018d58af04377b after 1ms
DEBUG(ClientCnxn.java:758)ClientCnxn:758 - Got ping response for sessionid: 0x1018d58af04377b after 1ms
DEBUG(ClientCnxn.java:758)ClientCnxn:758 - Got ping response for sessionid: 0x1018d58af04377b after 1ms
```

```shell
DEBUG(HeartBeatTask.java:66)HeartBeatTask:66 -  [DUBBO] Send heartbeat to remote channel /10.10.10.91:20809, cause: The channel has no data-transmission exceeds a heartbeat period: 60000ms, dubbo version: 2.4.7, current host: 192.168.4.215
DEBUG(HeartbeatHandler.java:82)HeartbeatHandler:82 -  [DUBBO] Receive heartbeat response in thread New I/O client worker #1-2, dubbo version: 2.4.7, current host: 192.168.4.215
DEBUG(HeartBeatTask.java:66)HeartBeatTask:66 -  [DUBBO] Send heartbeat to remote channel /10.10.10.46:20883, cause: The channel has no data-transmission exceeds a heartbeat period: 60000ms, dubbo version: 2.4.7, current host: 192.168.4.215
DEBUG(HeartbeatHandler.java:82)HeartbeatHandler:82 -  [DUBBO] Receive heartbeat response in thread New I/O client worker #1-3, dubbo version: 2.4.7, current host: 192.168.4.215
```

​	从日志上看能够看出来是dubbo和Zookeper的日志，原以为是日志级别的问题，就在logback.xml加上如下代码，将他们的日志设为error级别
```shell
<logger name="com.alibaba.dubbo" level="OFF" />
<logger name="org.apache.zookeeper.ClientCnxn" level="OFF" />

```
​	当然我上面是直接关了，你们可以看需求，但是然并卵。日志还是一如既往的刷刷刷~  我真的是脑子疼，项目启动慢就算了，你还如此冥王不顾，找死~

​	然后我就沉下心来，仔细排除，发现这尼玛不是我logback.xml中设置的log格式呀，你们是石头里蹦出来的吧。第一反应就是，这是log4j和slf4j同时使用了吧，难道是jar包问题？ 一顿顿操作啪啪啪，排除所有log4j jar包，重启~一如既往的刷刷刷~ 好吧，看来不是这个问题

3. 解决zookeeper日志问题
------


​	排查中发现
```shell
DEBUG(ClientCnxn.java:758)ClientCnxn:758 - Got ping response for sessionid: 0x1018d58af04377b after 1ms

```
这条日志中的‘ClientCnxn.class’源代码里面中的log包竟然是
```shell
import org.apache.log4j.Logger;
```
​	网上找到了答案，原来，zookeeper 3.3.3里的ClientCnxn的log直接用log4j，zookeeper 3.4.3里的ClientCnxn用的是slf4j。好吧，把zookeeper换成3.4.3,重启，果然这条日志不刷了


3. 解决dubbo日志问题
------

在启动项目的时候发现这条日志
```shell
INFO(?:?)LoggerFactory:? - using logger: com.alibaba.dubbo.common.logger.log4j.Log4jLoggerAdapter
```
怎么dubbo默认启动log4j的？只能去pom中找到dubbo 排除掉log4j架包~ 你以为就完了？重启，还是刷刷刷。。。

​	好吧，不行那只能看源码了，果然不看不知道，一看吓一跳
```shell
    static {
        String logger = System.getProperty("dubbo.application.logger");
        if ("slf4j".equals(logger)) {
            setLoggerAdapter((LoggerAdapter)(new Slf4jLoggerAdapter()));
        } else if ("jcl".equals(logger)) {
            setLoggerAdapter((LoggerAdapter)(new JclLoggerAdapter()));
        } else if ("log4j".equals(logger)) {
            setLoggerAdapter((LoggerAdapter)(new Log4jLoggerAdapter()));
        } else if ("jdk".equals(logger)) {
            setLoggerAdapter((LoggerAdapter)(new JdkLoggerAdapter()));
        } else {
            try {
                setLoggerAdapter((LoggerAdapter)(new Log4jLoggerAdapter()));
            } catch (Throwable var6) {
                try {
                    setLoggerAdapter((LoggerAdapter)(new Slf4jLoggerAdapter()));
                } catch (Throwable var5) {
                    try {
                        setLoggerAdapter((LoggerAdapter)(new JclLoggerAdapter()));
                    } catch (Throwable var4) {
                        setLoggerAdapter((LoggerAdapter)(new JdkLoggerAdapter()));
                    }
                }
            }
        }

    }
```

​	原来如果没有设置输出日志类型，dubbo中默认的日志是log4j，这这这，哎哟，我今天真的是脑壳疼了。网上答案是说在启动的时候设置System.getProperty("dubbo.application.logger")这个参数为slf4j

​	好吧直接上代码
```shell
public class Slf4jConfig extends ContextLoaderListener {

    @Override
    public void contextInitialized(ServletContextEvent event) {
        System.setProperty("dubbo.application.logger","slf4j");
    }
}
```

​	web.xml
```shell
    <listener>
        <listener-class>com.galaxy.wap.core.Slf4jConfig</listener-class>
    </listener>
```

​	启动，完成，果然源码的重要性立马凸显了









