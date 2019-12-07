---
title: 关于Aspect和Annotation的使用
tags: aspect
categories: java
---
#   关于Aspect和Annotation的使用

##  简述  
   昨儿碰到一个问题，某张表数据状态突然被更新了，据说不是人为，那应该就是天祸，故让我来排查下问题。  
   查了记录和日志没有什么有用的，翻了代码找到很多入口但不知道是哪触发的呀，链路追踪好像可以用zipkin，但现在还是说说一个简单的方法
   <!--more-->
   利用@Aspect和自定义注解监听需要监控的更新方法，然后？获取监控字段和请求堆栈记录下来。也就是我们常说的利用切面自定义注解实现日志记录，是一样的道理
 
 
##  自定义注解  

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FieldChangeAnnotation {

    /**
     * 需要监控字段
     * @return
     */
    String fieldName();

    /**
     * 保存表名称
     * @return
     */
    String entity();

    /**
     * 保存主键
     * @return
     */
    String primaryKey() default "id";
}
```
   
##  定义Aspect切面  

```
    @Aspect
    @Component
    public class UpdateAspect {
    
        @Autowired
        private FieldChangeLogManager fieldChangeLogManager;
    
        //第一个*是全部 第二个*是所有子包和子包下的类，updateBy*是监听以updateBy开头的方法，(..)是任意参数 
        @Before("execution(* com.galaxy.crm.business.impl..*.updateBy*(..)) && @annotation(fca)")
        public void before(JoinPoint join, FieldChangeAnnotation fca){
            //获取参数列表
            List<Object> args = Arrays.asList(join.getArgs());
    
            if (null != fca) {
                String fieldName = fca.fieldName();
    
                FieldChangeLog fieldChangeLog = new FieldChangeLog();
                fieldChangeLog.setEntityName(fca.entity());
                fieldChangeLog.setCreateTime(new Date());
                fieldChangeLog.setFieldName(fieldName);
                fieldChangeLog.setCreator(ThreadLocalHelper.getUserName());
    
                try {
                    args.stream().forEach(e -> {
                        Object value = getFieldValueByName(fieldName, e);
                        Object id = getFieldValueByName(fca.primaryKey(), e);
                        if (null != value && null != id){
                            String method = getCallClassName();
                            fieldChangeLog.setCallClassName(method);
                            fieldChangeLog.setFieldUpdate(String.valueOf(value));
                            fieldChangeLog.setPrimaryKey(String.valueOf(id));
                            fieldChangeLogManager.insert(fieldChangeLog);
                        }
                    });
                } catch(Exception e) {
    
                }
            }
        }
    
        //利用异常获取调用堆栈
        private String getCallClassName() {
            return ExceptionUtils.getStackTrace(new Throwable());
        }
    
        //获取Object中field字段值
        private Object getFieldValueByName(String fieldName, Object o) {
            try {
                String firstLetter = fieldName.substring(0, 1).toUpperCase();
                String getter = "get" + firstLetter + fieldName.substring(1);
                Method method = o.getClass().getMethod(getter, new Class[] {});
                Object value = method.invoke(o, new Object[] {});
                return value;
            } catch (Exception e) {
                return null;
            }
        }
    }
```