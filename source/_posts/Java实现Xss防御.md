---
title: Java防止Xss攻击
tags: Xss
categories: java
---

#	简述

什么是Xss攻击呢？就是Cross Site Scripting（跨站脚本攻击），缩写就是Xss。简单来说，就是恶意  
攻击者，利用存库等操作将Script代码存入数据库中，当有用户读取时执行，以达到攻击获取隐私数据  
的目的，本文为简单介绍防御方式。

<!--more-->

#	servlet防御

```
public class XssRequestWrapper extends HttpServletRequestWrapper {

    public XssRequestWrapper(HttpServletRequest servletRequest) {
        super(servletRequest);
    }
    @Override
    public String[] getParameterValues(String parameter) {
        String[] values = super.getParameterValues(parameter);
        if (values == null) {
            return null;
        }
        int count = values.length;
        String[] encodedValues = new String[count];
        for (int i = 0; i < count; i++) {
            encodedValues[i] = HtmlUtils.htmlEscape(values[i]);
        }
        return encodedValues;
    }
    @Override
    public String getParameter(String parameter) {
        String value = super.getParameter(parameter);
        return HtmlUtils.htmlEscape(value);
    }
    @Override
    public String getHeader(String name) {
        String value = super.getHeader(name);
        return HtmlUtils.htmlEscape(value);
    }
}
```

*   上面的HtmlUtils.htmlEscape() 我换成了下面的XssEncodeUtils,我个人比较喜欢
```
public class XssEncodeUtils {

    public static String stripXSS(String value) {
        if (value != null) {
            // NOTE: It's highly recommended to use the ESAPI library and uncomment the following line to
            // avoid encoded attacks.
            // value = ESAPI.encoder().canonicalize(value);

            // Avoid null characters
            value = value.replaceAll("", "");

            // Avoid anything between script tags
            Pattern scriptPattern = Pattern.compile("<script>(.*?)</script>", Pattern.CASE_INSENSITIVE);
            value = scriptPattern.matcher(value).replaceAll("");

            // Avoid anything in a src='...' type of e­xpression
            scriptPattern = Pattern.compile("src[\r\n]*=[\r\n]*\\\'(.*?)\\\'", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE | Pattern.DOTALL);
            value = scriptPattern.matcher(value).replaceAll("");

            scriptPattern = Pattern.compile("src[\r\n]*=[\r\n]*\\\"(.*?)\\\"", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE | Pattern.DOTALL);
            value = scriptPattern.matcher(value).replaceAll("");

            // Remove any lonesome </script> tag
            scriptPattern = Pattern.compile("</script>", Pattern.CASE_INSENSITIVE);
            value = scriptPattern.matcher(value).replaceAll("");

            // Remove any lonesome <script ...> tag
            scriptPattern = Pattern.compile("<script(.*?)>", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE | Pattern.DOTALL);
            value = scriptPattern.matcher(value).replaceAll("");

            // Avoid eval(...) e­xpressions
            scriptPattern = Pattern.compile("eval\\((.*?)\\)", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE | Pattern.DOTALL);
            value = scriptPattern.matcher(value).replaceAll("");

            // Avoid e­xpression(...) e­xpressions
            scriptPattern = Pattern.compile("e­xpression\\((.*?)\\)", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE | Pattern.DOTALL);
            value = scriptPattern.matcher(value).replaceAll("");

            // Avoid javascript:... e­xpressions
            scriptPattern = Pattern.compile("javascript:", Pattern.CASE_INSENSITIVE);
            value = scriptPattern.matcher(value).replaceAll("");

            // Avoid vbscript:... e­xpressions
            scriptPattern = Pattern.compile("vbscript:", Pattern.CASE_INSENSITIVE);
            value = scriptPattern.matcher(value).replaceAll("");

            // Avoid onload= e­xpressions
            scriptPattern = Pattern.compile("onload(.*?)=", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE | Pattern.DOTALL);
            value = scriptPattern.matcher(value).replaceAll("");
        }
        return value;
    }
}

```

#	Mybatis防御
```
public class XssStringTypeHandler extends BaseTypeHandler<String> {
    private static final Escaper XSS_ESCAPER = HtmlEscapers.htmlEscaper();

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        Pattern emoji = Pattern.compile ("[\ud83c\udc00-\ud83c\udfff]|[\ud83d\udc00-\ud83d\udfff]|[\u2600-\u27ff]",Pattern.UNICODE_CASE | Pattern . CASE_INSENSITIVE ) ;
        Matcher emojiMatcher = emoji.matcher(parameter);
        if ( emojiMatcher.find())
        {
            parameter = emojiMatcher.replaceAll("*");
        }
        ps.setString(i, XSS_ESCAPER.escape(parameter));
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return rs.getString(columnName);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return rs.getString(columnIndex);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
    }
}
```
*   mybatis-configuration.xml
typeHandlers中必须要添加需要过滤的字段类型
```
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
            "http://mybatis.org/dtd/mybatis-3-config.dtd">
    <configuration>
        <!-- 配置mybatis的缓存，延迟加载等等一系列属性 -->
        <settings>
            <!-- 全局映射器启用缓存 -->
            <setting name="cacheEnabled" value="false"/>
            <!-- 查询时，关闭关联对象即时加载以提高性能 -->
            <setting name="lazyLoadingEnabled" value="false"/>
            <!-- 设置关联对象加载的形态，此处为按需加载字段(加载字段由SQL指 定)，不会加载关联表的所有字段，以提高性能 -->
            <setting name="aggressiveLazyLoading" value="false"/>
            <!-- 对于未知的SQL查询，允许返回不同的结果集以达到通用的效果 -->
            <setting name="multipleResultSetsEnabled" value="true"/>
            <!-- 允许使用列标签代替列名 -->
            <setting name="useColumnLabel" value="true"/>
            <!-- 允许使用自定义的主键值(比如由程序生成的UUID 32位编码作为键值)，数据表的PK生成策略将被覆盖 -->
            <!-- <setting name="useGeneratedKeys" value="true" /> -->
            <!-- 给予被嵌套的resultMap以字段-属性的映射支持 -->
            <setting name="autoMappingBehavior" value="FULL"/>
            <!-- 对于批量更新操作缓存SQL以提高性能 -->
            <setting name="defaultExecutorType" value="REUSE"/>
            <!-- 数据库超过25000秒仍未响应则超时 -->
            <setting name="defaultStatementTimeout" value="25000"/>
        </settings>
    
        <typeHandlers>
            <typeHandler javaType="String" jdbcType="VARCHAR"
                         handler="com.galaxy.cobra.XssStringTypeHandler"/>
            <!--<typeHandler javaType="String" jdbcType="LONGVARCHAR"-->
                         <!--handler="com.galaxy.cobra.XssStringTypeHandler"/>-->
        </typeHandlers>
    </configuration>
```