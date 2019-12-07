---
title: SpringBoot的的自定义自动配置
tags: SpringBoot
categories: java
---

我想当我们提到SpringBoot的时候，可能大部分人都知道它有个好处就是自动配置，这里我就一个自定义的自动配置，
来简单记录下自动配置的原理吧。

<!-- more -->

##  原理

当我们启动Web项目的时候，都是从main方法开始，那么我们也从这个类开始看自动配置是如何实现的。在启动类的@SpringBootApplication注解里面，
我们重点看@EnableAutoConfiguration这个注解，意思是启动自动注解配置。
```
@SpringBootApplication  //<---
public class TestApplication {
	public static void main(String[] args) {
		SpringApplication.run(TestApplication.class, args);
	}
}

```

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration  // <---
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ···
}
```

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class) //<---
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```
而在@EnableAutoConfiguration注解中，我们看到这里import了一个类AutoConfigurationImportSelector.class
这里我们只需要知道这个类扫描了jar包路径下的META-INF/spring.factories文件，将文件中的内容包装成了properties文件对象
spring.factories内容类似如下：
```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener
```
而自动配置，就是通过加入容器的这些类来完成的

##  试一试

大致看了一下原理，那我们就来试着写一个自动配置类，首先在resources下创建文件META-INF/spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.platform.kangaroo.auto.TestAutoConfiguration
```
需要自动执行的类为com.platform.kangaroo.auto.TestAutoConfiguration,具体代码如下
```
@Configuration  //配置类
@EnableConfigurationProperties(TestProperty.class) //注入TestProperty的类
public class TestAutoConfiguration {

    /**
     * 若不使用EnableConfigurationProperties注解，也可以使用@Value注解获取properties内容
     */
    @Autowired
    private TestProperty testProperty;

    @Bean
    @ConditionalOnProperty(prefix = "edison",name = "name") //下面会讲@Conditional注解
    public void getMessage(){
        System.out.println(testProperty.getName());
    }
}
```
@EnableConfigurationProperties这个注解是与TestProperty类里面的@ConfigurationProperties配合来使用的，所以我们先看下这个类
```
@ConfigurationProperties(prefix = "edison")
public class TestProperty {
    private String name;

    private String address;

    private Integer age;

    public String getName() {
        return name;
    }

    public TestProperty setName(String name) {
        this.name = name;
        return this;
    }

    public String getAddress() {
        return address;
    }

    public TestProperty setAddress(String address) {
        this.address = address;
        return this;
    }

    public Integer getAge() {
        return age;
    }

    public TestProperty setAge(Integer age) {
        this.age = age;
        return this;
    }
}
```
上面的@ConfigurationProperties注解主要是用来把properties或yml文件转为bean的，而@EnableConfigurationProperties则是将这个类交给IOC容器，
这样你就能通过注解使用这个TestProperty类和内容了。

而@Conditional条件注解算是自动配置中比较重要的了，当使用这个注解的类，必须满足一定的条件才会装配Bean。这就是自动配置灵活的原因，它可以根据多种条件，
场景去决定要不要加载这个配置文件。而上面我的代码里面的@ConditionalOnProperty(prefix = "edison",name = "name")就是只有当properties文件中有edison.name
时才执行。
这里只是简单记录下这些类的说明：（具体很多都没使用过，所以不展示说明）
```
@ConditionalOnBean //仅当容器中含有特定的Bean，才会执行
@ConditionalOnClass //仅当classpath下含有特定class
@ConditionalOnCloudPlatform //给定的CloudPlatform必须是激活状态时
@ConditionalOnExpression    //表达式返回true时
@ConditionalOnJava //当JDK满足条件时
@ConditionalOnJndi //当Jndi接口可用
@ConditionalOnMissingBean   //当特有Bean不存在
@ConditionalOnMissingClass  //当classpath不存在特定class
@ConditionalOnNotWebApplication //当环境不是web时
@ConditionalOnProperty  //当配置文件含有特定值
@ConditionalOnResource  //当classpath下含有特定资源文件
@ConditionalOnSingleCandidate   //当只有一个指定bean
@ConditionalOnWebApplication    //当应用上下文为web环境时
```

当上面的都完成后，当我们在properties中配置内容edison.name=edison时，启动后进入getMessage()方法

##  总结
从上面的过程来看，在码代码的时候可能很少会用到，但是当需要增加一个中间件或者其他组件的时候自动配置可能是一个非常好的选择。