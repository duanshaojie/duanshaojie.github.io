---
title: FreeMarker的简单使用
tags: freeMarker
categories: java
---


1. 前言

------

​	FreeMarker是一款模板引擎: 即一种基于模板和要改变的数据， 并用来生成输出文本(HTML网页，电子邮件，配置文件，源代码等)的通用工具；通俗的讲，就是制定一个xml或者html的模板，进行数据填充的过程。

<!--more-->

​	模板编写为FreeMarker Template Language (FTL)。它是简单的，专用的语言， *不是* 像PHP那样成熟的编程语言。 那就意味着要准备数据在真实编程语言中来显示，比如数据库查询和业务运算， 之后模板显示已经准备好的数据。在模板中，你可以专注于如何展现数据， 而在模板之外可以专注于要展示什么数据。

​		我们看下这张官网的图片，也许你就知道FreeMarker是什么东西了。如下图，Template即是一个模板，${name}为需插入数据的标签。在Java代码中，向模板内set数据，即可获得output的数据

![](http://www.kerneler.com/freemarker2.3.23/figures/overview.png)

1. 环境

------

   pom文件需引入FreeMarker必要的架包，架包可以到这里找<https://mvnrepository.com>

```
   <dependency>
   	<groupId>org.freemarker</groupId>
   	<artifactId>freemarker</artifactId>
   	<version>2.3.23</version>
   </dependency>
```

   创建spring-beans.xml，添加FreeMarker配置，主要是FreeMarker模板的读取位置，也可以不注入spring-beans文件，在代码中设置读取位置

```
       <bean id="freeMarker" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">  
           <property name="templateLoaderPath" value="classpath:mail/" /> 
           <property name="freemarkerSettings"><!-- 设置FreeMarker环境属性 -->  
               <props>  
                   <prop key="template_update_delay">1800</prop> <!--刷新模板的周期，单位为秒 -->  
                   <prop key="default_encoding">UTF-8</prop> <!--模板的编码格式 -->  
                   <prop key="locale">zh_CN</prop> <!--本地化设置-->  
               </props>  
           </property>  
       </bean>
```

   spring-boot:创建一个.java文件，利用注解注入spring-beans.xml

```
   @Configuration
   @ImportResource(locations = "classpath:spring-beans.xml")
   public class DatabaseConfig {

   }
```

1. demo

------

   	接触FreeMarker主要是用作数据对接过程中的报文传输，这里的demo我使用FreeMarker模板发邮件。上面的spring-beans.xml中我们配置了，读取模板的位置。所以，我们将写好的模板放入mail文件夹内

```
   	src/main/resources

   		---mail

   			---register.ftl
```

```
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <title>邮件激活</title>
   </head>
   <body>
   <div>
       <p>亲爱的先生/女士：您好！</p>
       <p>&nbsp;&nbsp;&nbsp;&nbsp;您已成功注册成为XX会员,用户名为:<b style="font-family: 幼圆">${username}</b></p>
       <p>&nbsp;&nbsp;&nbsp;&nbsp;请在五分钟之内<a href=${url}>点击此处</a>进行激活。</p>
   </div>
   </body>
   </html>
```

   ​	怎么使用模板呢？我直接贴出我的代码，里面只需要注意获取Template的代码，和解析的代码就可~

```
   	@Autowired
       private FreeMarkerConfigurer freeMarkerConfigurer;
   	@Override
   	public void sendRegisterCode(String address) {
           try {
   			Template template = freeMarkerConfigurer.getConfiguration().getTemplate("register.ftl");
   			 // FreeMarker通过Map传递动态数据
   			String url = "duanshaojie.cn";
   	        Map<String, String> map = new HashMap<String, String>();   
   	        map.put("username", "edison_dsj@163.com"); 
   	        map.put("url", url); 
   	        String htmlText = FreeMarkerTemplateUtils.processTemplateIntoString(template, map); 
   	        send.sendMali("激活账号",address, htmlText, null, null);
   		} catch (Exception e) {
   			logger.error("EmailServiceImpl---register send Email error",e);
   		}    
           
   	}
```

1. freemarker模板内指令

------

   	在ftl模板文件中，在填充数据的过程中也可以使用一些简单的指令，用来辅助ftl的使用，这里介绍几个我用到的好了。

   ·值标签：${name}

​	日期格式:${birthdate?string('yyyy-MM-dd')}

​	数字格式:

​		${price?string.number} //20

​		${price?string.currency} //20.00

​		${price?string.percent} //20%

   ·注释:<#-- -->

   ·if语句:

```
   	<#if (1>0)>
   		<name>${name}</name>
   		<#else>
   		<name>大宝</name>	
   	</#if>
```

   ·判空:

```
       <#if (email)??>
       <email>${email}</email>
       </#if>
```

 ·遍历

```
		<#list person as p>
         <li>${p.name} 家在 ${p.address},电话是${p.mobile} 
         </#list>
```

宏(macro)使用，本来是没用到这玩意的。但是有人说，没使用过宏，就别说你用过FreeMarker，这就尴尬了，所以还是现学现卖弄，讲讲FreeMarker的宏；

​	那么啥是宏呢？

​	在FreeMarker中，宏就是和某些变量关联的模板片段，以便在模板中通过用户定义的指令使用该变量

​	①macro定义模板，调用

​	然后咋用呢？看个demo,首先定义

```
<#macro person> 
<font>Hello World!</font> 
</#macro>
```

然后使用:<@person/>

输出结果为:<font>Hello World!</font> 

​	②定义参数

```
<#macro person name> 
<font>Hello ${name}!</font> 
</#macro>
```

使用：<@person name="edc"/>

结果：<font>Hello edc!</font> 

​	③定义多参数：估计大部分人都猜得到，就是多个参数，参数可指定缺省值

```
<#macro person name info="hhhh"> 
<font>Hello ${name}!${info}</font> 
</#macro>
```

使用:<@person name="edc"/> 

结果:<font>Hello edc!hhhh</font> 

使用:<@person name="edc" info="哈哈"/>

结果:<font>Hello edc!哈哈</font>

​	④自定义指令嵌套内容<#nested> 

```
<#macro border>
<table border=4 cellspacing=0 cellpadding=4>
    <tr>
        <td>
            <#nested>
        </td>
    </tr>
</table>
</#macro>
```

使用

```
<@border>hello edc'blog</@border>
```

输出

```
<table border=4 cellspacing=0 cellpadding=4>
    <tr>
        <td>
            hello edc'blog
        </td>
    </tr>
</table>
```

`<#nested>`就相当于占位符

`<#nested>`指令可以被多次调用：

```
<#macro do>
    <#nested>
    <#nested>
    <#nested>
</#macro>
```

使用<@do>Anything.</@do>

输出

```
Anything.
Anything.
Anything.
```

