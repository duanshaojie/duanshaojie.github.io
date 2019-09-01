---
title: 9102年了，试试Webflux?
tags: springboot
categories: java
---

##   简述
最近新建了几个项目，但是发现搭来搭去感觉都可以使用项目模板。在小伙伴的推荐下，打算尝试下WebFlux，
人总是要尝试点新东西的嘛！这里简单记录下，好记性不如写篇博客~

<!-- more -->

##  什么是WebFlux？为啥用它

*   所谓webflux的就是Spring boot 2.0+以来新引入的基于Reactive Streams的框架。那什么是Reactive Streams呢？
    字面意思就是反应流，也就是基于事件而反应的管道数据（类似于Java8中的stream，数据在一条线上不断的被修改，最后返回结果）

*   我们这次选择用它，还有一个原因就是它是一种异步的数据流处理。其实我们在进行web开发时，会使用到的Tomcat容器，它实际是基于
Servlet运行的，而Servlet在3.0以前的请求是线程阻塞的，就是你的请求必须在获得所有的数据之后返回，而webflux是基于Netty容器

##  项目结构有啥区别？

*   项目大家可以在spring.io自动生成哈

*   我们以gradle为例，引入webflux包,这里不能引入spring-boot-strater-web包了,我们在原来的Web开发中使用的是javax.servlet.http
包下HttpServletRequest,和HttpServletResponse。而Webflux中使用的是org.springframework.http.server.reactive包下面的
ServerHttpResponse和ServerHttpRequest，当你重复引入的时候，ServerHttpResponse和ServerHttpRequest就不起作用了。
```
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
```

*   代码编写其实有两种方式，一种就是最为常见的@Controller,@Component层的编写，我下面将要写的就是Router,handler型的编写

##  一个简单的上手Router-仅记录代码
*   router-相当于controller
```
@Configuration
public class EdisonRouter {

    @Bean
    public RouterFunction<ServerResponse> hello(EdisonHandler edisonHandler) {
        return RouterFunctions.route(RequestPredicates.GET("/test/v1/hello")
        .and(RequestPredicates.accept(MediaType.TEXT_PLAIN)), edisonHandler::helloCity);
    }
}
```

*   handler-业务层：这里需要注意的是表示0个或一个返回结果的Mono,和返回多条结果的Flux,具体方法可以看看源码
```
@Component
public class EdisonHandler {

    public Mono<ServerResponse> helloCity(ServerRequest request) {
        return ServerResponse.ok().body(sayHelloCity(request), String.class);
    }

    private Mono<String> sayHelloCity(ServerRequest request) {
        String name = request.queryParam("name").orElseGet(() -> {
            throw new GlobalException(HttpStatus.INTERNAL_SERVER_ERROR, "name不能为空");
        });

        return Mono.just(name + "：hello ！");
    }
}
```

*   webflux filter
```
@Configuration
@Order(1)
public class CheckLoginFilter implements WebFilter {

    private static Set<String> unCheckUrl = new HashSet<>();

    static {
        unCheckUrl.add(SmsRouter.SEND_VERIFY_CODE);
    }

    @Autowired
    private Oauth2Manager oauth2Manager;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String token = request.getHeaders().getFirst("token");

        //如果包含则跳过
        if (containUnCheckUrl(request.getPath().toString())) {
            return chain.filter(exchange);
        }

        if (null == token) {
            throw new GlobalException(HttpStatus.UNAUTHORIZED, "token为空");
        }

        if (expiredToken(token)) {
            throw new GlobalException(HttpStatus.UNAUTHORIZED, "token已过期");
        }

        return chain.filter(exchange);
    }
}

```

##  碰到的问题
项目启动报
```
"Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]"

Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine a suitable driver class

//启动增加跳过源
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
```

图片上传接口
```
    @PostMapping("/file/v1/save")
        @ResponseBody
        public BaseResponse<String> save(Xxx xxx,
                                        @RequestPart("file") FilePart imageFile,

        }
```

图片下载接口
```
    @GetMapping("/file/v1/{fileName}")
    public Mono<Void> fileLoad(@PathVariable String fileName, ServerHttpResponse response) {
        try {
            //这里省略了获取文件的详细过程
            byte[] bytes = queryFileByte(fileName);
            DataBufferFactory dataBufferFactory = new DefaultDataBufferFactory();
            DataBuffer dataBuffer = dataBufferFactory.wrap(bytes);
            return zeroCopyResponse.writeWith(Flux.just(dataBuffer));
        } catch(Exception e) {
            logger.error("========== FileController fileLoad error={} ==========", e);
        }
        return null;
    }
```
或者使用webflux的函数式
```
//router:
@Configuration
public class QRcodeRouter {

    @Bean
    public RouterFunction<ServerResponse> createQrcode(QrcodeHandler qrcodeHandler){
        return RouterFunctions.route(RequestPredicates.GET("/qrcode/v1/create"), qrcodeHandler::create);
    }
}

//handler:
public Mono<ServerResponse> create(ServerRequest request) {
    String content = request.queryParam("content").orElseGet(() -> {
        throw new GlobalException("生成二维码参数为空");
    });
    ByteArrayOutputStream bytes = new ByteArrayOutputStream();
    QRCodeUtil.createCodeToOutputStream(SCHEMA + content, bytes);

    DataBufferFactory dataBufferFactory = new DefaultDataBufferFactory();
    DataBuffer dataBuffer = dataBufferFactory.wrap(bytes.toByteArray());

   return ServerResponse.ok()
            .body((p, a) -> {
                ZeroCopyHttpOutputMessage zeroCopyResponse = (ZeroCopyHttpOutputMessage) p;
                return zeroCopyResponse.writeWith(Flux.just(dataBuffer));
            });
}


```