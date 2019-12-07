---
title: Csrf防御
tags: csrf
categories: java
---

#	简述

CSRF（Cross Site Request Forgery, 跨站域请求伪造）是一种网络的攻击方式。

<!--more-->
#   攻击实例
CSRF 攻击可以在受害者毫不知情的情况下以受害者名义伪造请求发送给受攻击站点，从而在并未授权的情况下执行在权限保护之下的操作。比如说，受害者 Bob 在银行有一笔存款，通过对银行的网站发送请求 http://bank.example/withdraw?account=bob&amount=1000000&for=bob2 可以使 Bob 把 1000000 的存款转到 bob2 的账号下。通常情况下，该请求发送到网站后，服务器会先验证该请求是否来自一个合法的 session，并且该 session 的用户 Bob 已经成功登陆。黑客 Mallory 自己在该银行也有账户，他知道上文中的 URL 可以把钱进行转帐操作。Mallory 可以自己发送一个请求给银行：http://bank.example/withdraw?account=bob&amount=1000000&for=Mallory。但是这个请求来自 Mallory 而非 Bob，他不能通过安全认证，因此该请求不会起作用。这时，Mallory 想到使用 CSRF 的攻击方式，他先自己做一个网站，在网站中放入如下代码： src=”http://bank.example/withdraw?account=bob&amount=1000000&for=Mallory ”，并且通过广告等诱使 Bob 来访问他的网站。当 Bob 访问该网站时，上述 url 就会从 Bob 的浏览器发向银行，而这个请求会附带 Bob 浏览器中的 cookie 一起发向银行服务器。大多数情况下，该请求会失败，因为他要求 Bob 的认证信息。但是，如果 Bob 当时恰巧刚访问他的银行后不久，他的浏览器与银行网站之间的 session 尚未过期，浏览器的 cookie 之中含有 Bob 的认证信息。这时，悲剧发生了，这个 url 请求就会得到响应，钱将从 Bob 的账号转移到 Mallory 的账号，而 Bob 当时毫不知情。等以后 Bob 发现账户钱少了，即使他去银行查询日志，他也只能发现确实有一个来自于他本人的合法请求转移了资金，没有任何被攻击的痕迹。而 Mallory 则可以拿到钱后逍遥法外。

#   防御方式
##  验证 HTTP Referer
根据 HTTP 协议，在 HTTP 头中有一个字段叫 Referer，它记录了该 HTTP 请求的来源地址。
代码如下
```
@Configuration
public class LoginFilterBean {
    /**
     * csrf 拦截
     * @return
     */
    @Bean
    public FilterRegistrationBean filterRefer() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new RefererFilter());
        registration.addUrlPatterns("/*");
        registration.setName("referFilter");
        registration.setEnabled(true);
        //filter 排序
        registration.setOrder(10);

        return registration;
    }
}
```

```
    public class RefererFilter implements Filter {

    private final static Logger logger = LoggerFactory.getLogger(RefererFilter.class);

    private static final String EXCLUDED_URL_KEY = "excluded_url_str";
    private static final String REFERER_FILTER = "referer_filter";
    private static final String REFERER_PATTERN = "referer_pattern";
    private static final String TRUE = "true";

    private static String excluded;
    private static String refererStatus;
    private static String pattern;

    static {
        excluded = CrmConfig.getConfigVal(EXCLUDED_URL_KEY);
        refererStatus = CrmConfig.getConfigVal(REFERER_FILTER);
        pattern = CrmConfig.getConfigVal(REFERER_PATTERN);
    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest servletRequest = (HttpServletRequest) request;

        if (StringUtils.isNotBlank(refererStatus) && TRUE.equals(refererStatus)) {
            String referer = servletRequest.getHeader("Referer");

            if (StringUtils.isNotBlank(referer)) {
                Pattern p = Pattern.compile(pattern);
                Matcher m = p.matcher(referer);
                if (!m.find())
                {
                    logger.error("========== RefererFilter doFilter error referer={} ==========", referer);
                    throw new RuntimeException("referer valite error");
                }
            }else{
                String requestUrl = servletRequest.getRequestURL().toString();
                if (!isWhite(requestUrl)){
                    logger.error("========== RefererFilter doFilter error requestUrl={} ==========", requestUrl);
                    throw new RuntimeException("request url not exist");
                }
            }
        }
        chain.doFilter(request, response);
    }

    private boolean isWhite(String requestUrl) {
        if (StringUtils.isNotBlank(excluded)) {
            List<String> params = Arrays.asList(excluded.split("，"));
            for (String param : params) {
                if (requestUrl.indexOf(param) > -1) {
                    return true;
                }
            }
        }
        return false;
    }

    @Override
    public void destroy() {

    }
```

##  请求增加token验证
