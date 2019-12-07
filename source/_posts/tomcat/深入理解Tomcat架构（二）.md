---
title: 深入理解Tomcat架构（二）
tags: tomcat
categories: 中间件
---

##   简述

   在《深入理解Tomcat架构（一）》中，我们知道了tomcat架构中的主要两块，分别是Connector和Container，
而本文我们主要记录下Connector的三种运行模式。

<!--more-->

##  JAVA的I/O模型

    什么是I/O？I/O就是计算机内存与外部数据相互拷贝的过程。在介绍几种I/O模型之前，我们必须要知道在数据拷贝过程中，主要
    发生了什么？用户在等待内核从网卡拷贝数据；内核将数据从内核拷贝至用户空间。

*   同步阻塞I/O：当用户线程调起read时，该线程暂时挂起，而这个时候CPU就让给内核进行从网卡拷贝数据至内核，
一直到内核将数据拷贝至用户空间，才会唤醒线程

*   同步非阻塞I/O：而非阻塞的与阻塞的区别在于，线程可以多次调用read，但在数据还未拷贝至内核时，调用返回都是失败的，
只有当数据到达内核空间时，线程才会挂起，一直到数据从内核拷贝至用户空间，才会唤醒线程

*   I/O多路复用：这里的多路复用也叫事件驱动，前两个模型都是在用户调用read的阻塞或者返回错误，而多路复用则将请求分为了两步，
使用select监控到达内核的数据（select可以同时监控多个管道，所以称为多路复用），然后再调用read，保证每次read都能够拿到数据，
但是当从内核拷贝数据到用户空间的时候同样是阻塞的。

*   异步I/O：异步说明该模型是非阻塞的，当用户调用read的时候立马会获得返回，而但数据成功从网卡拷贝至内核时，会调用指定的回调函数
完成。

##  tomcat的三种运行模式

### BIO

*   BIO（blocking I/O), 就是阻塞式I/O，是基于JAVA的HTTP/1.1连接器，且tomcat7以下默认为BIO

### NIO

*   NIO(non-blocking I/O), 就是非阻塞I/O，tomcat8以上为默认NIO，配置位置在conf/server.xml中
```
<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
               connectionTimeout="20000"
               redirectPort="8443" />
```

### APR

*   APR(Apache Portable Runtime),APR与NIO一样都是非阻塞I/O，区别在于NIO使用的是JAVA的NIO来实现的，而APR则是利用JNI(Java Native Interface)
来调用C语言编写的APR。使用如下（对于Tomcat 7.0.30开始向后的版本，只需要再次修改protocol即可）
```
<Connector port="8080" protocol="org.apache.coyote.http11.Http11AprProtocol"
               connectionTimeout="20000"
               redirectPort="8443" />
```

##  tomcat如何扩展Java线程池

*   创建线程池
```
    /**
    *   corePoolSize 核心线程数
    *   maximumPoolSize 最大线程数
    *   keepAliveTime 超时时间，当临时线程从队列无法拉任务时，销毁
    */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }
```

*   一般线程池处理逻辑
```
小于corePoolSize个任务时，每来一个就创建一个线程处理；
但线程数大于corePoolSize时，则把线程放入队列，如果队列满了则创建临时线程处理；
而如果总线程数达到maximumPoolSize时，执行拒绝策略
```

*   tomcat扩展线程则重写了execute，更改了逻辑
```
小于corePoolSize个任务时，每来一个就创建一个线程处理；
但线程数大于corePoolSize时，则把线程放入队列，如果队列满了则创建临时线程处理；
而如果总线程数达到maximumPoolSize时，尝试将线程放入队列；
如果队列也满了，则执行拒绝策略

```

*   tomcat更改线程池配置，在conf/server.xml中
```
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
        maxThreads="150" minSpareThreads="4"/>

```


