---
title: 获取properties和spring单元测试
tags: java
categories: work
---

-  ### 背景

  ​	在java开发过程中，想必绝不部分人都使用过properties文件，比如数据库的配置信息，一些数据对接的信息等等。本文没有什么特别的难点，仅仅是记录下properties文件的使用，防止自己忘记，毕竟这个文件使用频次不高，我记性又不好 : -)

  <!--more-->

- ### 环境介绍

  spring boot 1.4.3.RELEASE

  jdk 1.8

  eclipse 5.4

  maven 3.3.9

  因为我仅仅是进行测试，所以这里需要引入spring-test，根据你的环境选择pom

  ```
  <!--spring-boot-->
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-test</artifactId>
  	<scope>test</scope>
  </dependency>
  ```

  ```
  <!--spring mvc-->
  <dependency>
  	<groupId>org.springframework</groupId>
  	<artifactId>spring-test</artifactId>
  	<version>${spring.version}</version>
  </dependency>
  ```

- ### 实现部分

  在开始之前，你需要先创建一个springboot项目，结构如下

  ```
  src/main/java
     ---cn.duanshaojie.app
  	Application.java
  src/main/test
     ---cn.duanshaojie.pro
  	---propertiesTest.java
  src/main/resources
     ---test.properties
  pom.xml
  ```

  具体项目如下

  pom.xml

  ```
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  	<modelVersion>4.0.0</modelVersion>

  	<groupId>cn.duanshaojie</groupId>
  	<artifactId>pro</artifactId>
  	<version>0.0.1-SNAPSHOT</version>
  	<packaging>jar</packaging>

  	<name>pro</name>
  	<url>http://maven.apache.org</url>

  	<!-- 继承父包 -->
  	<parent>
  		<groupId>org.springframework.boot</groupId>
  		<artifactId>spring-boot-starter-parent</artifactId>
  		<version>1.4.3.RELEASE</version>
  	</parent>

  	<properties>
  		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  	</properties>

  	<dependencies>
  		<!-- spring-boot-starter-web依赖 -->
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-web</artifactId>
  		</dependency>
  		<dependency>
  			<groupId>junit</groupId>
  			<artifactId>junit</artifactId>
  			<version>3.8.1</version>
  			<scope>test</scope>
  		</dependency>
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-test</artifactId>
  			<scope>test</scope>
  		</dependency>
  	</dependencies>
  </project>
  ```

  test.properties

  ```
  test.url=duanshaojie.com
  test.username=duanshaojie
  ```

  Application.java

  我们看到这里有两个标签

  @SpringBootApplication：

  ​	它包括三个注解：(这三个注解一般是一起使用的，spring-boot把它们合为了@SpringBootApplication)

  ​	@Configuration：表示将该类作用springboot配置文件类。

  ​	@EnableAutoConfiguration:表示程序启动时，自动加载springboot默认的配置。

  ​	@ComponentScan:表示程序启动是，自动扫描当前包及子包下所有类

  ​	有了@SpringBootApplication只是默认的，你也可以重写这三个注解，并配置你想要更改的属性~

  @PropertySource：该注解的使用可以获取properties中的key-value值,也可以同时注入多个properties文		件

  ```
  @SpringBootApplication
  @PropertySource(value = { "classpath:test.properties" })
  public class Application {
  	public static void main(String[] args) {
          SpringApplication.run(Application.class, args);
  	}
  }
  ```

  到这里环境就准备的差不多了，接下来我们来写测试

  首先是已知key，我们直接注入，然后使用@Value("${}")标签获取key的值，这里用到的几个标签都是spring和JUnit单元测试时使用的标签，讲到这里干脆在后面再具体点的记一下好了

  ```
  //propertiesTest.java
  @RunWith(SpringJUnit4ClassRunner.class)
  @SpringBootTest(classes = cn.duanshaojie.app.Application.class) 
  public class propertiesTest {

  @Value("${test.url}")
  private String url;

  @Test
  public void getUrlTest(){
  	System.out.println(url);
  }
  }
  ```

  输出:duanshaojie.com

  还有一种是我们也不知道key到底是哪一个,是根据业务逻辑获得key，然后再来properties中获取

  ```
  @RunWith(SpringJUnit4ClassRunner.class)
  @SpringBootTest(classes = cn.duanshaojie.app.Application.class) 
  public class propertiesTest {

  @Autowired
  private ConfigurableEnvironment  env;

  @Test
  public void getUrlTest(){
  	System.out.println(env.getProperty("test.user"));
  }
  }
  ```

- ### 单元测试

 - #### SpringBoot单元测试

  我们上面都使用了spring-test，那么啥是spring-test框架呢？在解释之前，我们先看看，spring如果要测试，用传统的Junit是怎么测试的：

  1.建立一个hello类

  ```
  public class Hello{
  	public void sayHello(){
        System.out.println("你好啊");
  	}
  }
  ```

  2.编写测试类

  ```
  public class junitTest {
  	@Test
  	public void say(){
  		ApplicationContext  res = new 	ClassPathXmlApplicationContext("classpath:applicationContext.xml");
  		Hello h = (Hello) res.getBean("hello");
  		h.sayHello();
  	}
  }
  ```

  输出：你好啊

  ​

  这里从网上找了传统JUnit在spring测试过程中的问题

  - 问题一：导致Spring 容器多次初始化，性能开销很大。
  - 问题二：不应该由测试代码管理Spring容器，应该是由Spring容器来管理测试代码。
  - 问题三：无法独立于服务器完成事务测试等。

  ​

  而spring-test框架就出现了

  - 使用Spring Test 有助于减少启动容器的开销，提高测试效率。
  - Spring Test可以直接使用@AutoWired注入Spring容器或bean。
  - Spring Test还支持事务测试，集成测试等。

  在springboot的使用spring-test十分的便捷，只需要你在pom文件中引入架包

  ```
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-test</artifactId>
  	<scope>test</scope>
  </dependency>
  ```

  我们看刚才进行properties测试的代码拿来看一下就知道，单元测试只需要加上如下的两个标签就可以了

  1.@RunWith(SpringJUnit4ClassRunner.class)

  ​	表示整合JUnit测试

  2.@SpringBootTest(classes = cn.duanshaojie.app.Application.class) 

  ​	我们的spring-boot的启动会去加载很多的配置文件，这里我们的注解就可以引用那些配置文件

  在SpringBoot1.4以前，这个注解还没有出现的时候，是使用**@SpringApplicationConfiguration**(classes=)和

  **@WebAppConfiguration**代替的

  ​

-  #### Spring单元测试

这里准备一个Util，继承就可以使用@Autowired了

```
@RunWith(SpringJUnit4ClassRunner.class)
@Configuration
@WebAppConfiguration
@ContextConfiguration(locations = { "classpath:spring-beans.xml" })
public class BaseUnit {

}
```

