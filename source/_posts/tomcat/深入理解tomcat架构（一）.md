---
title: 深入理解Tomcat架构（一）
tags: tomcat
categories: 中间件
---

   在工作中，我虽然经常使用Tomcat部署项目，却没有深入理解它的架构原理，这篇文章算是记录下自己学习的成果。
同时也希望你读完这篇文章能够对tomcat的架构原理能有一定的了解~~ :-)
<!--more-->

##   为什么要学习tomcat？ 
   我们都知道，当你用浏览器请求一个链接的时候，简单的过程就是DNS解析-TCP连接-HTTP请求-获取响应。而这个过程中的HTTP
是如何调用你所编写的Controller的呢？其实tomcat就相当于一个容器，所有的Http请求都与Tomcat连接，并把请求交给tomcat，
由tomcat去解析，交给相应的Controller

##   什么是Servlet？  
   为什么这里要先介绍Servlet呢，tomcat就是基于Servlet实现的一个Servlet容器。而一个最简单的Servlet是什么样的呢？
```
public class EdisonServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException
    {
        System.out.println("myServlet 在处理get()请求...");
        PrintWriter out = resp.getWriter();
        resp.setContentType("text/html;charset=utf-8");
        out.println("<strong>My get servlet!</strong><br>");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        System.out.println("myServlet 在处理post()请求...");
        PrintWriter out = resp.getWriter();
        resp.setContentType("text/html;charset=utf-8");
        out.println("<strong>My post servlet!</strong><br>");
    }
}
```
其实只需要继承HttpServlet，重写相应的doGet和doPost方法就行，这个地方其实可以全手写一个Servlet，不用
到HttpServlet，有时间可以试试。

##   Tomcat的架构  
   其实从Tomcat的conf目录下server.xml可以看出来，Tomcat相当于一个Server，而一个Server中有多个service,而一个service中
主要包含多个connector和一个container组件，后面会解释server.xml内的各个节点含义。

*   Connector连接器  
    连接器的作用实际上就是替container容器接受所有请求（例如HTTP，AJP等），相当于一个中转吧，将请求转为标准的ServletRequest
    ```
    监听网络端口
    接受读取网络请求
    生成容器能够识别的，通用的ServletRequest
    调用容器获得ServletResponse，转为tomcat Response对象
    返回给浏览器
    ```
    而完成这些过程的主要组件为：Endpoint,processor和Adapter.
    Endpoint和processor又抽象成为一个ProtocolHandler
    *   Endpoint  
        Endpoint主要用来进行通信。（实现TCP/IP）
        
    *   processor  
        processor主要用来处理请求，从Endpoint读取字节流转换为Tomcat能够识别的Request或者Response对象
    
    *   Adapter  
        转换Tomcat的Request和Response对象转为ServletRequest和ServletResponse
        
*   Container容器  
    *   tomcat内有四种父子关系的容器,可以通过server.xml加深理解 
    ```
    |_Engine  
       |__Host(代表一个虚拟主机)  
          |__Context(表示一个web应用程序)     
             |__Wrapper(代表一个servlet)
    ```
    *   那么一个请求是如何定位到一个Servlet的呢？  
        当一个请求来临的时候，Mapper组件解析了URL的域名和路径，再根据自己保存的wrapper映射路径，定位到一个Servlet
    

##  Tomcat中的server.xml
   这里是网上看到，借用记录下
   ```
   1、Server提供一个接口，由1至多个Service组成，让其它程序可以访问到这个Service集合，同时维护各个Service的生命周期，包括如何初始化，如何结束服务，如何找到别人请求的服务。
   
   2、Service又由1-n个Connector及单个Container组成,只是在Container和Connector外多包了一层，提供各种服务
   
   3、Connector组件是可选择替换的，负责接收浏览器发过来的TCP连接请求，创建Request/Response,分配线程，将创建的对象传递给Container来处理请求
   
   4、Container是容器的父接口，由四个容器组成，分别是Engine，Host，Context，Wrapper。其中Engine包含Host，Host包含Context，Content包含Wrapper，一个Servlet class对应一个Wrapper
   
   5、Engine容器是作为顶级Container组件来设计的，由Host组成,其作用相当于一个Container的门面。有了Engine，请求的处理过程变为：浏览器发出请求，Connector接受请求，将请求交由Container（这里是Engine）处理，Container（Engine来担当）查找请求对应的Host并将请求传递给它，Host拿到请求后查找相应的应用上下文环境，准备 servlet环境并调用service方法。
   
   6、Host容器是Engine的子容器，一个Host在Engine中代表一个虚拟主机，这个主机可以运行多个应用，他负责安装和展开这些应用，并且标识这个应用，以便能够区分他们。它的子容器通常是Context，他除了关联子容器外，还保存一个主机应有的信息。Host不是必需的，但是要运行 war程序，就必须要使用Host，因为在war中必有web.xml文件，这个文件解析就需要Host，如果有多个Host就需要定义一个top容器 Engine，而Engine没有父容器，一个Engine就代表一个完整的Servlet引擎
   
   7、Context代表Servlet的Context，它具备Servlet运行的基本环境，理论上只要有Context就能运行Servlet，简单的Tomcat可以没有Engine和Host。其最重要的功能就是管理Servlet实例，Servlet实例在Context中是以Wrapper 出现的。
   
   8、Wrapper代表一个Servlet，它负责管理Servlet，包括装载，初始化，执行以及资源回收。它是最底层的容器。
   ```
