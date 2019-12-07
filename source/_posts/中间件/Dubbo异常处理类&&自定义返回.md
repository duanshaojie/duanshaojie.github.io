---
title: Dubbo异常处理类&&自定义返回
tags: dubbo
categories: 中间件
---

   起初是因为对外SDK群里有人反映，接口报了系统错误，不知道具体是为什么。后经排查是因为dubbo接口的参数校验等，都是用RuntimeException进行抛出的，
   而SDK针对Exception仅仅是捕获并对外报“系统错误”，那在不改动sdk的情况下，只有针对dubbo接口抛出的runtimeException做一个统一异常处理了。
   而且我们的dubbo接口所有的返回对象都不是统一的，最后怎么解决的呢，请往后看..

<!-- more -->

#  准备
## 版本
```
compile ('org.apache.dubbo:dubbo:2.7.2')
```

## 本地调试

需要在消费端，在需要本地调用的接口增加本地dubbo-url，端口为提供方配置“dubbo.protocol.port=21799”
```xml
<dubbo:reference id="extendProductApiService" interface="com.xxx.xxx.service.ExtendProductApiService"
                     check="false" url="dubbo://localhost:21799" />
```

# 异常处理类

dubbo提供了基于SPI的Filter功能，而且也有一个ExceptionFilter异常处理类了。那么我们既然要使用这个异常类，
说明它并不能满足我们的需求，需要针对特定业务覆盖这个类。
我们可以在源码的**MATA-INF/dubbo/internal**下发现这个文件**org.apachedubbo.rpc.Filter**，针对dubbo的filter
链问题，有兴趣的同学也可以关注下。

## 重写方式

而我们覆盖这个类的方式有两个：
*   我们看到**org.apachedubbo.rpc.Filter**这个文件中有这么一行（如下图），它指定了异常处理的类路径，而我们可以创建
resources/META-INF/dubbo/internal/org.apachedubbo.rpc.Filter。
```text
cache=org.apache.dubbo.cache.filter.CacheFilter
validation=org.apache.dubbo.validation.filter.ValidationFilter
echo=org.apache.dubbo.rpc.filter.EchoFilter
generic=org.apache.dubbo.rpc.filter.GenericFilter
genericimpl=org.apache.dubbo.rpc.filter.GenericImplFilter
token=org.apache.dubbo.rpc.filter.TokenFilter
accesslog=org.apache.dubbo.rpc.filter.AccessLogFilter
activelimit=org.apache.dubbo.rpc.filter.ActiveLimitFilter
classloader=org.apache.dubbo.rpc.filter.ClassLoaderFilter
context=org.apache.dubbo.rpc.filter.ContextFilter
consumercontext=org.apache.dubbo.rpc.filter.ConsumerContextFilter
exception=org.apache.dubbo.rpc.filter.ExceptionFilter           //<<<---------------这里
executelimit=org.apache.dubbo.rpc.filter.ExecuteLimitFilter
deprecated=org.apache.dubbo.rpc.filter.DeprecatedFilter
compatible=org.apache.dubbo.rpc.filter.CompatibleFilter
timeout=org.apache.dubbo.rpc.filter.TimeoutFilter
trace=org.apache.dubbo.rpc.protocol.dubbo.filter.TraceFilter
future=org.apache.dubbo.rpc.protocol.dubbo.filter.FutureFilter
monitor=org.apache.dubbo.monitor.support.MonitorFilter

metrics=org.apache.dubbo.monitor.dubbo.MetricsFilter

```
而org.apachedubbo.rpc.Filter这个文本文件我们只保留需要覆盖的,
````text
exception=com.xxx.xxx.config.ExceptionFilter
````
并且复制ExceptionFilter类到相应目录下：
```java
@Activate(group = CommonConstants.PROVIDER)
public class ExceptionFilter extends ListenableFilter {

    public ExceptionFilter() {
        super.listener = new ExceptionListener();
    }

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

    static class ExceptionListener implements Listener {

        private Logger logger = LoggerFactory.getLogger(ExceptionListener.class);

        @Override
        public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
            if (appResponse.hasException() && GenericService.class != invoker.getInterface()) {
                try {
                    Throwable exception = appResponse.getException();

                    // directly throw if it's checked exception
                    if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
                        return;
                    }
                    // directly throw if the exception appears in the signature
                    try {
                        Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                        Class<?>[] exceptionClassses = method.getExceptionTypes();
                        for (Class<?> exceptionClass : exceptionClassses) {
                            if (exception.getClass().equals(exceptionClass)) {
                                return;
                            }
                        }
                    } catch (NoSuchMethodException e) {
                        return;
                    }

                    // for the exception not found in method's signature, print ERROR message in server's log.
                    logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

                    // directly throw if exception class and interface class are in the same jar file.
                    String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface());
                    String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
                    if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
                        return;
                    }
                    // directly throw if it's JDK exception
                    String className = exception.getClass().getName();
                    if (className.startsWith("java.") || className.startsWith("javax.")) {
                        return;
                    }
                    // directly throw if it's dubbo exception
                    if (exception instanceof RpcException) {
                        return;
                    }

                    // otherwise, wrap with RuntimeException and throw back to the client
                    appResponse.setException(new RuntimeException(StringUtils.toString(exception)));
                    return;
                } catch (Throwable e) {
                    logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                    return;
                }
            }
        }

        @Override
        public void onError(Throwable e, Invoker<?> invoker, Invocation invocation) {
            logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
        }

        // For test purpose
        public void setLogger(Logger logger) {
            this.logger = logger;
        }
    }
}

```

*   第二种覆盖方式，就是根据org.apachedubbo.rpc.Filter文件，我们可知ExceptionFilter所在的路径。而我们只需要在包内创建一样的路径和文件就能够进行覆盖。

##  ExceptionFilter的拦截方式和问题
可以对照源码的那几行注释看：
*   直接抛异常的情况
    *   受检异常直接抛出
    *   dubbo方法上有明确throws的异常
    *   异常类与接口所在包一致
    *   异常类路径以javax.和java.开头的（即JDK异常）
    *   dubbo自己的异常

*   除上面的情况以外，其他的异常会转为runtimeException再抛出。
__这里就会有个问题，就是自定义的异常，消费端就不能直接抓到处理，因为都转成了runtimeException__，这里也有几种方式：
    *   针对自定义异常处理
    ```java
    if(exception instanceof 自定义异常) {
        return;
    }
    ```
    *   在接口中明确抛出
    *   将自定义异常与接口放在一个包内

#  返回自定义对象
除了直接抛出异常，还可以针对异常做统一的自定义对象返回：
## 针对返回值一致的情况
更改ExceptionFilter类中的onResponse方法如下：
```
        public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
            if (appResponse.hasException() && GenericService.class != invoker.getInterface()) {
                try {
                    Throwable exception = appResponse.getException();
                    //省略...
                    //这里可根据自己业务的逻辑进行返回，DubboResponse为自定义的对象
                    if (exception instanceof RuntimeException) {
                        DubboResponse response = new DubboResponse();
                        response.setMsg(response.getMsg());
                        appResponse.setValue(response);
                        appResponse.setException(null);//注意这里的exception要置为空，否则无效
                        return;
                    }

                    // otherwise, wrap with RuntimeException and throw back to the client
                    appResponse.setException(new RuntimeException(StringUtils.toString(exception)));
                    return;
                } catch (Throwable e) {
                    logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                    return;
                }
            }
        }
```

## 针对返回值都不一致情况
就如我开头所说，我们的返回对象都不一致，导致不能如此简单的返回。其实这里也很简单，我们可以通过获取className，
反射生成需要返回的实例，省略后的代码如下，**重点在获取返回className那哦~~~**
```
    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
```
```java
        /**
         * @author edison
         */
        @Activate(group = CommonConstants.PROVIDER)
        public class ExceptionFilter extends ListenableFilter {

            public ExceptionFilter() {
                super.listener = new ExceptionListener();
            }

            @Override
            public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
                return invoker.invoke(invocation);
            }

            static class ExceptionListener implements Listener {

                private Logger logger = LoggerFactory.getLogger(ExceptionListener.class);

                @Override
                public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
                    if (appResponse.hasException() && GenericService.class != invoker.getInterface()) {
                        try {
                            Throwable exception = appResponse.getException();

                            if (exception instanceof RuntimeException) {
                                try {
                                    Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                                    if (null != method) {
                                        String className = method.getReturnType().getName();
                                        Object obj = conversionObj(className, exception.getMessage());
                                        appResponse.setValue(obj);
                                        appResponse.setException(null);
                                        return;
                                    }else{
                                        logger.error("========== ExceptionFilter handleException getMethod is null ==========");
                                    }
                                } catch (Exception ex) {
                                    logger.error("========== ExceptionFilter conversionObj error ==========", ex);
                                }
                            }

                            // otherwise, wrap with RuntimeException and throw back to the client
                            appResponse.setException(new RuntimeException(StringUtils.toString(exception)));
                            return;
                        } catch (Throwable e) {
                            logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                            return;
                        }
                    }
                }

                private Object conversionObj(String className, String message) throws ClassNotFoundException, IllegalAccessException, InstantiationException, NoSuchFieldException {
                    Class class_ = Class.forName(className);
                    Object obj = class_.newInstance();
                    Field msg = class_.getSuperclass().getDeclaredField("msg");
                    Field result_ = class_.getSuperclass().getDeclaredField("result");
                    Field errorCode = class_.getSuperclass().getDeclaredField("errorCode");
                    msg.setAccessible(true);
                    result_.setAccessible(true);
                    errorCode.setAccessible(true);
                    msg.set(obj, message);
                    result_.set(obj, false);
                    errorCode.set(obj, "400");
                    return obj;
                }

                @Override
                public void onError(Throwable e, Invoker<?> invoker, Invocation invocation) {
                    logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
                }

                // For test purpose
                public void setLogger(Logger logger) {
                    this.logger = logger;
                }
            }
        }

```

#  这里在记一下还有另一种方式
```java
/**
 * @author edison
 */
@Activate(group = CommonConstants.PROVIDER)
public class ExceptionFilter extends ListenableFilter  {

    private final static Logger logger = LoggerFactory.getLogger(ExceptionFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        Result result = invoker.invoke(invocation);
        if (result.hasException()) {
            return this.handleException(result, invocation);
        }
        return result;
    }


    /**
     * 异常处理
     * @param result
     * @param invocation
     * @return
     */
    private Result handleException(Result result, Invocation invocation) {
        Throwable e = result.getException();
        if (e instanceof RuntimeException) {
            logger.error("dubbo统一异常捕获 --> runtimeException", e);
            //返回解析
            AsyncRpcResult rpcResult = new AsyncRpcResult(invocation);
            try {
                Object obj = conversionObj(invocation, e.getMessage());
                rpcResult.setValue(obj);
                return rpcResult;
            } catch (NoSuchFieldException | InstantiationException | IllegalAccessException | ClassNotFoundException ex) {
                logger.error("========== ExceptionFilter conversionObj error ==========", ex);
            }
        }
        return result;
    }

    /**
     * 解析
     * @param invocation
     * @param message
     * @return
     * @throws IllegalAccessException
     * @throws ClassNotFoundException
     * @throws NoSuchFieldException
     * @throws InstantiationException
     */
    private Object conversionObj(Invocation invocation, String message) throws IllegalAccessException, ClassNotFoundException, NoSuchFieldException, InstantiationException {
        //获取返回对象
        String className = ((RpcInvocation) invocation).getReturnType().getName();
        Class class_ = Class.forName(className);
        Object obj = class_.newInstance();
        Field msg = class_.getSuperclass().getDeclaredField("msg");
        Field result_ = class_.getSuperclass().getDeclaredField("result");
        Field errorCode = class_.getSuperclass().getDeclaredField("errorCode");
        msg.setAccessible(true);
        result_.setAccessible(true);
        errorCode.setAccessible(true);
        msg.set(obj, message);
        result_.set(obj, false);
        errorCode.set(obj, "400");
        return obj;
    }
}

```

##     碰到的问题

*   dubbo2.7.2以后RpcResult没有了，详见<a href="https://github.com/apache/dubbo/issues/4282" target="_blank">《升级到2.7.2后，怎么没有RpcResult类了？用什么代替呢？》</a>
*   java.lang.UnsupportedOperationException:AppResponse represents an concrete business response, there will be no status changes, you should get internal values directly.详见<a href="https://github.com/apache/dubbo/issues/4717" target="_blank">《dubbo》</a>