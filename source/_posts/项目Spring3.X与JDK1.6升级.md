---
title: 项目Spring3.X与JDK1.6升级
tags: jdk
categories: java
---

# 项目Spring3.X与JDK1.6升级

##  简述

*   环境 spring:3.2.7.RELEASE jdk:1.6

*   其主要问题是解决一些包冲突，和包版本升级

<!-- more -->

##  升级

*   由于spring3.X 不兼容JDK 1.8，故需升级spring到4.X
    本例升级spring为4.3.7.RELEASE
    ```
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring.version>4.3.7.RELEASE</spring.version>
    </properties>
    ```
    
*   升级 org.ow2.asm:asm -> 5.X
    ```
    <dependency>
        <groupId>org.ow2.asm</groupId>
        <artifactId>asm</artifactId>
        <version>5.0.4</version>
     </dependency>
    ```
    
*   Spring-web.xml中
    ```
    <bean id="mappingJacksonHttpMessageConverter" class="org.springframework.http.converter.json.MappingJacksonHttpMessageConverter">
    
    MappingJacksonHttpMessageConverter->>MappingJackson2HttpMessageConverter
    ```
    
*   升级 redis.clients:jedis -> 2.9.0
    ```
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>2.9.0</version>
    </dependency>
    ```
    而jedis 依赖的commons-pool 需要排除，重新引入commons-pool2
    ```
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
        <version>2.4.2</version>
    </dependency>
    ```
    升级jedis2.9信息变更
    -“maxActive” -> “maxTotal”
    -“maxWait” -> “maxWaitMillis”
    ```
    <bean id="jedisPool" class="redis.clients.jedis.JedisPool" destroy-method="destroy">
        <constructor-arg index="0">
            <ref bean="jedisPoolConfig"/>
        </constructor-arg>
        <constructor-arg index="1" value="shopping.redis.master.galaxy.com" type="java.lang.String"  />
        <constructor-arg index="2" value="6379" type="int"/>
    </bean>
    ```
    JedisPool 需增加类型
    
*   spring4.X 在dubbo注入与本地已有类名服务时报错，如下，为javassist版本过低

*   更新XML中spring.XXX.xsd 版本为4.0