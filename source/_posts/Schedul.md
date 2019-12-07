---
title: 记录下Java定时任务-Schedule入门使用
tags: 定时任务
categories: java
---

schedule:
	最近在工作中碰到需要使用定时任务的需求，所以在此记录下，仅贴出简单demo

<!-- more -->

### java.util.Timerd 定时任务

------

Timer是一个定时调度类,Timer是调度控制器，而TimerTask是可调度的任务，任务的执行方式有两种：（具体不会细说，直接上代码）

- ##### 固定延迟：timer.schedule(task, delay)---task为任务；delay延迟时间（毫秒）

  ```
  public class TaskTest {
  	public static void main(String[] args) {
  		Timer timer = new Timer();
  		System.out.println(new Date());
  		timer.schedule(new TimerTask() {
  			
  			@Override
  			public void run() {
  				System.out.println(new Date()+",延迟任务,2秒后执行,只执行一次");
  			}
  		}, 2000);
  	}
  }
  ```

  输出如下

  ```
  Wed Aug 23 16:05:04 CST 2017
  Wed Aug 23 16:05:06 CST 2017,延迟任务,2秒后执行,只执行一次
  ```

- ##### 固定速率：

  ```
  public class TaskTest {
  	public static void main(String[] args) {
  		Timer timer = new Timer();
  		System.out.println(new Date());
  		timer.schedule(new TimerTask() {
  			
  			@Override
  			public void run() {
  				System.out.println(new Date()+",延迟任务,2秒后开始任务(每秒1次)");
  			}
  		}, 2000,1000);
  	}
  }
  ```

  输出如下：

  ```
  Wed Aug 23 16:08:52 CST 2017
  Wed Aug 23 16:08:54 CST 2017,延迟任务,2秒后开始任务(每秒1次)
  Wed Aug 23 16:08:55 CST 2017,延迟任务,2秒后开始任务(每秒1次)
  Wed Aug 23 16:08:56 CST 2017,延迟任务,2秒后开始任务(每秒1次)
  ```

  从以上的例子，顺便从网上偷了个图片如下，如有侵权，联系删除

  ![Timer](http://images.cnblogs.com/cnblogs_com/jinspire/201202/201202101400252596.png)

  Timer的多任务是有一定缺点的，这里不具体解释，仅做入门

### 环境配置

------

因为任务调度是一个单独的服务，所以这里使用的是springboot，springboot的使用请异步

[spring-boot]: http://projects.spring.io/spring-boot/

- ##### 先创建一个maven项目

- ##### 目录结构

  src/main/java

  ​	cn.duanshaojie.app

  ​		--Application.java

  ​	cn.duanshaojie.config

  ​		--QuartzConfig.java

  ​	cn.duanshaojie.task

  ​		--LogTask.java

  src/main/resources

  ​	--job

  ​		--schedul.xml

- ##### 贴上的我pom文件

  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.duanshaojie</groupId>
    <artifactId>schedule</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>schedule</name>
    <url>http://maven.apache.org</url>

    <parent>
    	<groupId>org.springframework.boot</groupId>
    	<artifactId>spring-boot-starter-parent</artifactId>
    	<version>1.4.3.RELEASE</version>
    </parent>

    <properties>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
      <dependency>
    		<groupId>org.springframework.boot</groupId>
    		<artifactId>spring-boot-starter-web</artifactId>
    	</dependency>
    	<dependency>
      	<groupId>org.springframework.boot</groupId>
      	<artifactId>spring-boot-starter-redis</artifactId>
      </dependency>
      <dependency>
      	<groupId>org.springframework.boot</groupId>
      	<artifactId>spring-boot-starter-aop</artifactId>
      </dependency>
          <dependency>
      	<groupId>org.quartz-scheduler</groupId>
      	<artifactId>quartz</artifactId>
      	<version>2.2.3</version>
      </dependency>
      <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>3.8.1</version>
        <scope>test</scope>
      </dependency>
    </dependencies>
  </project>
### 具体操作

------

- ##### Application.java

  ```
  @SpringBootApplication
  @ComponentScan(value = "cn.duanshaojie")
  public class Application {
  	public static void main(String[] args) {
          SpringApplication.run(Application.class, args);
  	}
  }
  ```

  ​

- ##### QuartzConfig.java

  ```
  @Configuration
  @ImportResource(locations = { "classpath:job/schedul.xml" })
  public class QuartzConfig {

  }
  ```

  ​

- ##### LogTask.java

  ```
  @Component("logTask")
  public class LogTask {
  	private static final Logger logger = LoggerFactory.getLogger(LogTask.class);
  	
  	public void executeTask(){
  		Thread curren = Thread.currentThread();
  		System.out.println("定时任务:"+curren.getId());
  		logger.info("定时任务2:"+curren.getId()+",name"+curren.getName());
  	}
  }

  ```

  ​

- ##### Schedul.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd">
	
	<bean id="logDetail"
		class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
		<!--false表示等上一个任务执行完后再开启新的任务 -->
		<property name="concurrent" value="false" />
		<property name="targetObject">
			<ref bean="logTask" />
		</property>
		<property name="targetMethod">
			<value>executeTask</value>
		</property>
	</bean>

	<!-- 调度触发器 -->
	<bean id="logTrigger"
		class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
		<!-- <property name="name" value="work_default_name" /> <property name="group" 
			value="work_default" /> -->
		<property name="jobDetail">
			<ref bean="logDetail" />
		</property>
		<property name="cronExpression">
  			<value>0/1 * * * * ?</value>
		</property>
	</bean>

	<!-- 调度工厂 -->
 	<bean id="schedulerTrigger"
		class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
		<property name="triggers">
			<list>
				<ref bean="logTrigger" />
			</list>
		</property>
	</bean>
</beans>
```



在做完demo之后，发现还有一种更简单的schedul调度，就是注解式的

只是在以上的代码中做些修改，在Application启动文件中加上注解@EnableScheduling扫描schedul方法

而调度任务可以这样写

```
public class LogTask {
	private static final Logger logger = LoggerFactory.getLogger(LogTask.class);
	
	@Scheduled(cron="0/1 * * * * ?")
	public void executeTask2(){
		Thread curren = Thread.currentThread();
		System.out.println("定时任务:"+curren.getId());
		logger.info("定时任务1:"+curren.getId()+",name"+curren.getName());
	}
}
```

### CronTrigger时间配置

------

格式: [秒][分] [小时][日] [月][周] [年]

0 0 12 * * ?           每天12点触发 
0 15 10 ? * *          每天10点15分触发 
0 15 10 * * ?          每天10点15分触发  
0 15 10 * * ? *        每天10点15分触发  
0 15 10 * * ? 2005     2005年每天10点15分触发 
0 * 14 * * ?           每天下午的 2点到2点59分每分触发 
0 0/5 14 * * ?         每天下午的 2点到2点59分(整点开始，每隔5分触发)  
0 0/5 14,18 * * ?        每天下午的 18点到18点59分(整点开始，每隔5分触发)

0 0-5 14 * * ?            每天下午的 2点到2点05分每分触发 
0 10,44 14 ? 3 WED        3月分每周三下午的 2点10分和2点44分触发 
0 15 10 ? * MON-FRI       从周一到周五每天上午的10点15分触发 
0 15 10 15 * ?            每月15号上午10点15分触发 
0 15 10 L * ?             每月最后一天的10点15分触发 
0 15 10 ? * 6L            每月最后一周的星期五的10点15分触发 
0 15 10 ? * 6L 2002-2005  从2002年到2005年每月最后一周的星期五的10点15分触发

0 15 10 ? * 6#3           每月的第三周的星期五开始触发 
0 0 12 1/5 * ?            每月的第一个中午开始每隔5天触发一次 
0 11 11 11 11 ?           每年的11月11号 11点11分触发(光棍节)

- ##### 好吧，我们来看几个例子

  0 0 12 * * ? 每天12点触发 
  0 15 10 ? * * 每天10点15分触发 
  0 15 10 * * ? 每天10点15分触发 
  0 15 10 * * ? * 每天10点15分触发 
  0 15 10 * * ? 2005 2005年每天10点15分触发 
  0 * 14 * * ? 每天下午的 2点到2点59分每分触发 
  0 0/5 14 * * ? 每天下午的 2点到2点59分(整点开始，每隔5分触发) 
  0 0/5 14,18 * * ? 每天下午的 2点到2点59分(整点开始，每隔5分触发) 
  每天下午的 18点到18点59分(整点开始，每隔5分触发) 
  0 0-5 14 * * ? 每天下午的 2点到2点05分每分触发 
  0 59 2 ? * FRI    每周5凌晨2点59分触发； 
  0 15 10 ? * MON-FRI 从周一到周五每天上午的10点15分触发 
  0 15 10 15 * ? 每月15号上午10点15分触发 
  0 15 10 L * ? 每月最后一天的10点15分触发 
  0 15 10 ? * 6L 每月最后一周的星期五的10点15分触发 
  0 15 10 ? * 6L 2002-2005 从2002年到2005年每月最后一周的星期五的10点15分触发 
  0 15 10 ? * 6#3 每月的第三周的星期五开始触发 
  0 0 12 1/5 * ? 每月的第一个中午开始每隔5天触发一次

好了，调度的简单使用就是上面这些了，有什么问题和意见，请发我邮箱 ：)