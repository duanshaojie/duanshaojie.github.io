---
title: 手写JDK动态代理
tags: java
categories: java
---

我们知道JDK的动态代理十分的强大，它能够根据一个类，在不改动类源码的情况下，使用一个代理类添加属于自己的逻辑，并重新生成一个新的类。那么它是如何实现的呢？我也不知道，所以这篇文章让我们一起学习下它的实现原理，跟着动手写一个自己的动态代理。

<!-- more -->
##  静态代理
可能大家都听说过设计模式中的代理模式，实际上静态代理的定义就是，使用一个代理类，来实现对某些类的访问权限。
比如这样一个场景，有一天我要把我的房子租出去,但是我不想和那么多陌生人有交集，那我们就找中介呗，如下：
*   建一个房源抽象接口
```java
//房源
public interface Listings {
    void getMessage();
}
```
*   我的房源信息
```java
public class EdisonRentMessage implements Listings {

    @Override
    public void getMessage() {
        System.out.println("房主：edison；月租金：3000");;
    }
}
```
*   中介为我的房源做了代理
```java
public class ProxyRentMessage implements Listings {

    @Override
    public void getMessage() {
        System.out.println("我是XX房屋经纪人");
        new EdisonRentMessage().getMessage();
    }
}
```
*   租客查看时，其实只能看到中介，并不知道背后房源的真实地址
```java
public class Demo {
    public static void main(String[] args) {
        Listings listings = new ProxyRentMessage();
        listings.getMessage();
    }
}
```
```
//打印
我是XX房屋经纪人
房主：edison；月租金：3000
```
##  动态代理
动态代理与静态代理有什么区别呢？
*   静态代理在编译后已经是一个.class文件被加载到jvm了。而动态代理则是在运行过程中，动态生成并加入到JVM中的，并没有实体.class。
*   由于代理类需要在运行时加载字节码，所以相比较静态代理，效率就没有那么高
但是静态代理又会有个问题，每个代理类都只对一个业务接口进行包装，如果现在这个代理兼职卖房我们怎么修改呢？
（并且我现在需要他每个代理接口前先上个广告，结束以后再加个结束语。)

```java
public class ProxyRentMessage implements Listings {

    //租房
    @Override
    public void getMessage() {
        System.out.println("我是XX房屋经纪人,欢迎您查看我们公司的房源");
        new EdisonRentMessage().getMessage();
        System.out.println("以上就是房源，若喜欢就和我联系吧。");
    }

    //卖房信息
    @Override
    public void getSellMessage() {
        System.out.println("我是XX房屋经纪人,欢迎您查看我们公司的房源");
        new EdisonRentMessage().getSellMessage();
        System.out.println("以上就是房源，若喜欢就和我联系吧。");
    }
}
```
那如果代理多了，以上的代码就会越来越臃肿，这个时候就可以使用动态代理，下面根据JDK的动态代理修改了代理类：
```java
public class RentInvocation implements InvocationHandler {

    private Object target;

    public <T extends Listings> Object getInstance(T target) {
        this.target = target;
        Class<?> clazz = target.getClass();
        return Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("我是XX房屋经纪人,欢迎您查看我们公司的房源");
        Object result = method.invoke(target, args);
        System.out.println("以上就是房源，若喜欢就和我联系吧。");
        return result;
    }
}
```
调用端及输出如下：
```java
public class Demo {
    public static void main(String[] args) {
        Listings listings = new EdisonRentMessage();
        Listings listings2 = (Listings) new RentInvocation().getInstance(listings);
        listings2.getMessage();
    }
}
```
```text
我是XX房屋经纪人,欢迎您查看我们公司的房源
房主：edison；月租金：3000
以上就是房源，若喜欢就和我联系吧。
```
一个动态代理，首先实现InvocationHandler类并在invoke方法中进行业务操作，而一个动态代理也需要维护一个target对象。

##  手写实现JDK动态代理
*   首先在手写JDK动态代理之前，我们来缕一下他生成新对象的过程：
    *   获取被代理对象的引用，并且通过反射获取他所有的接口；
    *   重新生成一个类，并实现被代理类的所有接口；
    *   动态生成代码，添加需要加入的业务代码；
    *   编译生成Java的.class文件，并重新加载到JVM；

*   首先创建一个与InvocationHandle一样的接口
```java
public interface MyInvocationHandler {
    Object invoke(Object target, Method method, Object[] args) throws Throwable;
}
```

*   编写自定义ClassLoader类
```java
public class MyClassLoader extends  ClassLoader {

    private File classPathFile;

    public MyClassLoader(){
        String classpath = MyClassLoader.class.getResource("").getPath();
        this.classPathFile = new File(classpath);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String classpath = MyClassLoader.class.getPackage().getName() + "." + name;

        if (classPathFile.exists()) {
            File classFile = new File(classPathFile, name.replaceAll("\\.", "/") + ".class");
            if (classFile.exists()) {
                FileInputStream in = null;
                ByteArrayOutputStream out = null;

                try {
                    in = new FileInputStream(classFile);
                    out = new ByteArrayOutputStream();
                    byte[] bytes = new byte[1024];
                    int len;
                    while ((len = in.read(bytes)) != -1) {
                        out.write(bytes, 0, len);
                    }
                    return defineClass(classpath, out.toByteArray(), 0, out.size());
                } catch(Exception e) {
                    e.printStackTrace();
                }finally {
                    if (null != in) {
                        try {
                            in.close();
                        } catch(Exception e) {
                            e.printStackTrace();
                        }
                    }
                    if (null != out) {
                        try {
                            out.close();
                        } catch(Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
        return null;
    }
}
```
*   编写动态生成代理类的工具类
```java
public class ProxyCreateUtil {

    /**
     * 生成代理类工具
     * @param loader
     * @param interfaces    目标class的所有接口
     * @param invocationHandler 实例化需要生成的handler
     * @return
     */
    public static Object newProxyInstance(MyClassLoader loader, Class<?>[] interfaces, InvocationHandler invocationHandler) {
        try {
            //动态生成源代码
            String src = generateSrc(interfaces);
            //输出到磁盘
            String filepath = ProxyCreateUtil.class.getResource("").getPath();
            File file = new File(filepath + "$proxy0.java");
            FileWriter fileWriter = new FileWriter(file);
            fileWriter.write(src);
            fileWriter.flush();
            fileWriter.close();

            //将生成的.java编译成.class文件
            JavaCompiler javaCompiler = ToolProvider.getSystemJavaCompiler();
            StandardJavaFileManager manager = javaCompiler.getStandardFileManager(null,null,null);
            Iterable iterable = manager.getJavaFileObjects(file);
            JavaCompiler.CompilationTask task = javaCompiler.getTask(null, manager, null, null, null, iterable);
            task.call();
            manager.close();

            //将编译完成的.class放入JVM
            Class proxyClass = loader.findClass("$Proxy0");
            Constructor constructor = proxyClass.getConstructor(MyInvocationHandler.class);
            file.delete();

            //返回字节码充足以后的代理对象
            return constructor.newInstance(invocationHandler);
        } catch(Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    //生成源代码  <-------
    private static final String ln = "\r\n";
    private static String generateSrc(Class<?>[] interfaces) {
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append("package com.spring.invocation.InvocationHandler;" + ln);
        //静态代理时使用的接口
        stringBuffer.append("import com.spring.invocation._static.Listings;" + ln);
        stringBuffer.append("import java.lang.reflect.*;" + ln);

        stringBuffer.append("public class $Proxy0 implements " + interfaces[0].getName() + "{" + ln);
            stringBuffer.append("private MyInvocationHandler h;" + ln);
            stringBuffer.append("public $Proxy0(MyInvocationHandler h) {" + ln);
                stringBuffer.append("this.h = h" + ln);
            stringBuffer.append("}" + ln);

            //所有的方法
            for (Method m : interfaces[0].getMethods()) {
                Class<?>[] params = m.getParameterTypes();

                StringBuffer paramNames = new StringBuffer();
                StringBuffer paramValue = new StringBuffer();
                StringBuffer paramClasses = new StringBuffer();

                for (int i = 0; i < params.length; i++) {
                    Class clazz = params[i];
                    String type = clazz.getName();
                    String paramName = toLowerFirstCase(clazz.getSimpleName());

                    paramNames.append(type + " " + paramName);
                    paramValue.append(paramName);
                    paramClasses.append(clazz.getName() + ".class");

                    if (i > 0 && i < params.length - 1) {
                        paramNames.append(",");
                        paramValue.append(",");
                        paramClasses.append(",");
                    }
                }

                stringBuffer.append("public " + m.getReturnType().getName() + " " + m.getName() + "(" + paramNames.toString() +"){" + ln);
                    stringBuffer.append("try{" + ln);
                        stringBuffer.append("Method m = " + interfaces[0].getName() + ".class.getMethod(\""
                                        + m.getName() + "\", new Class[]{" + paramClasses.toString() + "});" + ln);
                        stringBuffer.append((hasReturnValue(m.getReturnType()) ? "return " : "") + getCaseCode("this.h.invoke(this,m,new Object[]{"
                                + paramValue + "})", m.getReturnType()) + ";" + ln);
                    stringBuffer.append("}catch(Error _ex){}");
                    stringBuffer.append("catch(Throwable e){" + ln);
                    stringBuffer.append("throw new UndeclaredThrowableException(e);" + ln);
                    stringBuffer.append("}");
                    stringBuffer.append(getReturnEmptyCode(m.getReturnType()));
                stringBuffer.append("}" + ln);
            }

        stringBuffer.append("}" + ln);
        return stringBuffer.toString();
    }

    private static Map<Class, Class> mappings = new HashMap<>();

    static {
        mappings.put(int.class, Integer.class);
    }

    private static String getReturnEmptyCode(Class<?> returnType) {
        if (mappings.containsKey(returnType)) {
            return "return 0;";
        }else if (returnType == void.class){
            return "";
        }else{
            return "return null;";
        }
    }

    private static String getCaseCode(String s, Class<?> returnType) {
        if (mappings.containsKey(returnType)) {
            return "((" + mappings.get(returnType).getName() + ")" + s + ")." + returnType.getSimpleName() + "value()";
        }
        return s;
    }

    private static boolean hasReturnValue(Class<?> returnType) {
        return returnType != void.class;
    }

    //生成首字母小写
    private static String toLowerFirstCase(String simpleName) {
        char[] chars = simpleName.toCharArray();
        chars[0] += 32;
        return chars.toString();
    }
}

```
*   最后参考静态代理编写测试方法
```java
    //生成动态代理类
public class DemoInvocationHandler implements MyInvocationHandler {

    private Object target;

    public Object getInstence(Object target) {
        this.target = target;
        Class clazz = target.getClass();
        return ProxyCreateUtil.newProxyInstance(new MyClassLoader(), clazz.getInterfaces(), this);
    }

    @Override
    public Object invoke(Object target, Method method, Object[] args) throws Throwable {

        doBefore();
        Object result = method.invoke(this.target, args);
        doAfter();
        return result;
    }

    private void doAfter() {
        System.out.println("以上就是房源，若喜欢就和我联系吧。");
    }

    private void doBefore() {
        System.out.println("我是XX房屋经纪人,欢迎您查看我们公司的房源");
    }
}

```
*   测试类及返回如下
```java
public class Demo {
    public static void main(String[] args) {
        Listings listings = (Listings) new DemoInvocationHandler().getInstence(new EdisonRentMessage());
        listings.getMessage();
    }
}
```
```
我是XX房屋经纪人,欢迎您查看我们公司的房源
房主：edison；月租金：3000
以上就是房源，若喜欢就和我联系吧。
```

##  参考书籍
*   《Spring 5 核心原理与30个类手写实战》

