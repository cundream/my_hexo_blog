---
title:  Spring Cloud Alibaba-05-Gateway网关-03-过滤器(Filter)使用
date: 2025-04-05 21:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.10.20`

# Spring Cloud Alibaba-05-Gateway网关-03-过滤器(Filter)使用

[toc]



## 过滤器的简介

### 什么是过滤器？

* 过滤器就是在请求的传递中对请求和响应做一些手脚。
* 声明周期 Pre(处理前)，Post(处理后)

### 过滤器的生命周期

Gateway中，Filter的生命周期只有两个，pre和post，

* PRE :这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等
* POST：这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header，收集统计信息和指标、将响应从微服务发送到客户端。

![在这里插入图片描述](typora-user-images/1134316536b140eda97bef40993a2072.png)



### 过滤器的分类和作用范围

Gateway的Filter从作用范围可分为GatewayFilter与GlobalFliter

- GatewayFilter: 应用到某个路由或者一个分组的路由上。
- GlobalFilter: 应用到所有的路由上。



## 局部过滤器



### 内置局部过滤器

官方地址文档：https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gatewayfilter-factories

官方文档介绍了各种内置的局部过滤器，包含了案例。



![image-20240223110332038](typora-user-images/image-20240223110332038.png)



以 AddResponseHeader为例子，使用这个过滤器



~~~yaml
AddResponseHeader=X-Response-Red, Lison
~~~



![image-20240223110630296](typora-user-images/image-20240223110630296.png)



### 自定义局部过滤器

#### 实现自定义过滤器

需求配置Log,第一个参数控制台是否打印，第二个参数缓存日志是否打印

1. 先在配置文件中进行配置





~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         	- Method=GET
        filters: # 过滤器,请求在传递过程中可以通过过滤器对其进行一定的修改
          - StripPrefix=1 # 转发之前去掉1层路径
          - Log=true,false #开启缓存日志和控制台日志，第一个参数控制台日志，第二个参数缓存日志
 
 
~~~



2. 编写内部过滤器



~~~java
package com.lison.springcloudservice.config.filter;

import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.Arrays;
import java.util.List;


/**
 * @className: com.lison.springcloudservice.config.filter-> LogGatewayFilterFactory
 * @description: 自定义的局部过滤器 1. 类名必须是配置+GatewayFilterFactory 2. 必须继承AbstractGatewayFilterFactory
 * @author: Lison
 * @createDate: 2023-10-20
 */
@Component
public class LogGatewayFilterFactory extends AbstractGatewayFilterFactory<LogGatewayFilterFactory.Config> {

    public LogGatewayFilterFactory() {
        super(LogGatewayFilterFactory.Config.class);
    }

    //过滤器的逻辑
    @Override
    public GatewayFilter apply(Config config) {
        return new GatewayFilter() {
            @Override
            public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
                //获取封装 HttpRequest 后的对象
                ServerHttpRequest request = exchange.getRequest();
                //获取封装 HttpResponse 后的对象
                ServerHttpResponse response = exchange.getResponse();
                System.out.println("-------请求到来，经过全局 Filter 处理-------");
                if (config.isCacheLog()) {
                    System.out.println("cacheLog开启了-------");
                }
                if (config.isConsoleLog()) {
                    System.out.println("consoleLog开启了-------");
                }
                Mono<Void> mono = chain.filter(exchange);
                return mono;
            }
        };
    }

    //用于接收参数，需要和配置文件里的参数对应
    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList("consoleLog", "cacheLog");
    }

    public static class Config {
        private boolean consoleLog;
        private boolean cacheLog;

        public boolean isConsoleLog() {
            return consoleLog;
        }

        public void setConsoleLog(boolean consoleLog) {
            this.consoleLog = consoleLog;
        }

        public boolean isCacheLog() {
            return cacheLog;
        }

        public void setCacheLog(boolean cacheLog) {
            this.cacheLog = cacheLog;
        }

        @Override
        public String toString() {
            return "Config{" +
                    "consoleLog=" + consoleLog +
                    ", cacheLog=" + cacheLog +
                    '}';
        }
    }
}

~~~



3、启动网关测试，可以看到consoleLog打印了。

~~~
-------请求到来，经过局部 Log Filter 处理-------
consoleLog开启了-------
~~~





总结：自定义局部过滤器

1. 类名必须是配置+GatewayFilterFactory
2. 必须继承AbstractGatewayFilterFactory



## 全局过滤器

全局过滤器不和具体的路由关联，作用在所有路由上，不需要配置。通过全局过滤器可以实现对权限的统一校验，安全性校验等。



### 内置全局过滤器



![img](typora-user-images/779689.png)

我们只需要记住以下几个常用的即可

1. **AddRequestHeader**：给当前请求添加一个请求头，例如 - AddRequestHeader=color, red 就表示给请求头总加入一个键值对 color=red.

2. **AddRequestParameter**：表示给请求加一个参数，例如 - AddRequestParameter=name, cyk 也就意味着后端对应的接口因该提供一个参数去接收

3. AddResponseHeader：给响应结果中添加一个响应头，例如 - AddResponseHeader=name,cyk 那么后端在返回响应的时候就会在响应头中添加 name=cyk 的键值对.

4. PrefixPath：表示给请求的 url 中添加前缀，例如 - PrefixPath=/user ，那么如果你原本的请求是路径是 localhost:8080/list ，那么经过 PrefixPath 处理之后，就会变成localhost:8080/user/list

5. StripPrefix：表示给请求的 url 中去表指定的 n 个前缀路由，例如 - StripPrefix=2 那么如果你原本的请求是路由是 /user/list/get ，那么经过 StripPrefix 处理后，就会变成 /get.


> Spring提供了31种不同的路由过滤器工厂，这些需要什么都可以直接去官网上copy，不用刻意去记~



### 自定义全局过滤器



内置的过滤器已经实现了大部分的功能，但是对应企业开发的一些业务功能处理，还是需要我们自己编写过滤器来实现的。
开发中的鉴权逻辑

* 当客户端第一次请求服务时，服务端对用户进行认证(登录)
* 认证通过后，将用户信息进行加密形成token，返回给客户端，作为登录凭证
* 以后每次请求，客户端都携带认证的token
* 服务端对token进行解密，判断是否有效



#### 实现一个鉴权过滤器



~~~java
package com.lison.springcloudservice.config.filter;

import org.apache.commons.lang.StringUtils;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

/**
 * @className: com.lison.springcloudservice.config.filter-> AuthGlobalFilter
 * @description:  自定义鉴权全局过滤器  必须实现GlobalFilter和Ordered两个接口，并实现接口的方法
 * @author: Lison
 * @createDate: 2023-10-20
 */
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    //过滤器逻辑
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
//获取封装 HttpRequest 后的对象
        ServerHttpRequest request = exchange.getRequest();
        //获取封装 HttpResponse 后的对象
        ServerHttpResponse response = exchange.getResponse();
        System.out.println("-------请求到来，经过全局 Filter 处理-------");
        //对请求做处理......
        Mono<Void> filter = chain.filter(exchange);//放行 filter 向后执行：也就意味接下来进入到具体的微服务中(例如 user 微服务)
        //对响应做处理......
        System.out.println("-------响应回来，经过全局 Filter 处理-------");

        //将token放在参数里面传递只是一个演示，正常的业务逻辑当然不是这样做，这里只是为了演示。
        String token = request.getQueryParams().getFirst("token");
        if(!StringUtils.equals("admin",token)){
            System.out.println("认证失败");
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);//401表示未认证
            return exchange.getResponse().setComplete();
        }

        //chain.filter继续传递向下
        return filter;
    }

    //返回值越小，优先级越高
    @Override
    public int getOrder() {
        return 0;
    }
}
~~~



2、启动网关测试，可以看到日志打印了。

~~~
http://localhost:18003/spring_service/getServerProd

-------请求到来，经过全局 Filter 处理-------
-------响应回来，经过全局 Filter 处理-------
认证失败


http://localhost:18003/spring_service/getServerProd?token=admin
-------请求到来，经过全局 Filter 处理-------
-------响应回来，经过全局 Filter 处理-------
-------请求到来，经过局部 Log Filter 处理-------
consoleLog开启了-------
~~~



总结：

1. 实现 GlobalFilter 接口，重写 filter 方法
2. 添加 @Order 注解或实现 Ordered 接口
3. 编写处理逻辑





## 过滤器执行顺序

请求进入网关会碰到三类过滤器（当前路由的过滤器、DefaultFilter、GlobalFilter），然后合并到一个过滤器集合中，排序后一次执行每一个过滤器，那么他的一个执行顺序是怎样的呢？

执行规则

1.order值越小，优先级越高（GlobalFilter由我们自定义，路由过滤器和defaultFilter的order由Spring指定，默认是按照声明顺序从1递增）

2.当order值一样时，顺序是defaultFilter > 路由过滤器 > 全局过滤器。



## Gateway 跨域问题处理

跨域：域名不一致就是跨域，主要包括：

域名不同： www.taobao.com 和 www.taobao.org 和 www.jd.com 和 miaosha.jd.com
域名相同，端口不同：localhost:8080和localhost8081
跨域问题：浏览器禁止请求的发起者与服务端发生跨域ajax请求，请求被浏览器拦截的问题

解决方案：CORS

> 网关处理跨域采用的同样是CORS方案，并且只需要简单配置即可实现（这些东西不用记住，需要的时候 CV 就可以）



~~~yaml
spring:
  cloud:
    gateway:
      globalcors: # 全局的跨域处理
        add-to-simple-url-handler-mapping: true # 解决options请求被拦截问题
        corsConfigurations:
          '[/**]':
            allowedOrigins: # 允许哪些网站的跨域请求(2.4版本以后需要更改为 allowedOriginPatterns ) 
              - "http://localhost:18001"
              - "http://www.taobao.com"
            allowedMethods: # 允许的跨域ajax的请求方式
              - "GET"
              - "POST"
              - "DELETE"
              - "PUT"
              - "OPTIONS"
            allowedHeaders: "*" # 允许在请求中携带的头信息
            allowCredentials: true # 是否允许携带cookie
            maxAge: 360000 # 这次跨域检测的有效期（为例减少性能损耗，在有效时间内，浏览器将不在发起询问，直接放行通过）
~~~

当然也可以不用分的这么细，可以直接用下面整个通用的配置即可.



~~~yaml
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedMethods: "*"
            allowedHeaders: "*"
            allowedOriginPatterns: "*"
            allowCredentials: true
~~~

