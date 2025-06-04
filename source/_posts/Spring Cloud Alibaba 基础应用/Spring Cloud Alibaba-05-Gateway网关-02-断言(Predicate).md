---
title: Spring Cloud Alibaba-05-Gateway网关-02-断言(Predicate)使用
date: 2025-04-05 20:10:56
---

`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.10.20`

# Spring Cloud Alibaba-05-Gateway网关-02-断言(Predicate)使用



[toc]



Predicate 断言，用于进行条件判断，只有断言都为真，才会真正的执行路由。



## 通过时间匹配

Predicate 支持设置一个时间，在请求进行转发的时候，可以通过判断在这个时间之前或者之后进行转发。比如我们现在设置只有在 2023年 10 月 20 日才会转发到我的网站，在这之前不进行转发，我就可以这样配置：



~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - After=2023-10-20T08:30:00+08:00[Asia/Shanghai]
 
 
~~~

Spring 是通过 ZonedDateTime 来对时间进行的对比，ZonedDateTime 是 Java 8 中日期时间功能里，用于表示带时区的日期与时间信息的类，ZonedDateTime 支持通过时区来设置时间，中国的时区是：Asia/Shanghai。

After Route Predicate 是指在这个时间之后的请求都转发到目标地址。上面的示例是指，请求时间在 2023 年 10 月 20 日 08 点 30 分 0 秒之后的所有请求都转发到地址http://localhost:18001。+08:00是指时间和 UTC 时间相差八个小时，时间地区为Asia/Shanghai。

添加完路由规则之后，访问地址http://localhost:18001/spring_service 会自动转发到http://localhost:18001。

Before Route Predicate 刚好相反，在某个时间之前的请求的请求都进行转发。我们把上面路由规则中的 After 改为 Before，如下:



~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Before=2023-10-20T08:30:00+08:00[Asia/Shanghai]
 
 
~~~



就表示在这个时间之前可以进行路由，在这时间之后停止路由，修改完之后重启项目再次访问地址`http://localhost:18001/spring_service `，页面会报 404 没有找到地址。

除过在时间之前或者之后外，Gateway 还支持限制路由请求在某一个时间段范围内，可以使用 Between Route Predicate 来实现

~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Between=2023-10-20T08:30:00+08:00[Asia/Shanghai],2024-10-20T08:30:00+08:00[Asia/Shanghai]
 
 
~~~

这样设置就意味着在这个时间段内可以匹配到此路由，超过这个时间段范围则不会进行匹配。通过时间匹配路由的功能很酷，可以用在限时抢购的一些场景中。

## 通过 Cookie 匹配



~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Cookie=token,lison
 
 
~~~



![image-20240222175326047](typora-user-images/image-20240222175326047.png)

![image-20240222175403127](typora-user-images/image-20240222175403127.png)

**总结：去掉Cookie或Cookie不正确,后台汇报 404 错误。带上正确的Cookie正常访问**





## 通过 Header 匹配

Header Route Predicate 和 Cookie Route Predicate 一样，也是接收 2 个参数，一个 header 中属性名称和一个正则表达式，这个属性值和正则表达式匹配则执行。



~~~java
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Header=X-Request-Id, \d+
 
 
~~~



![image-20240222180227522](typora-user-images/image-20240222180227522.png)

![image-20240222180306582](typora-user-images/image-20240222180306582.png)

![image-20240222180325402](typora-user-images/image-20240222180325402.png)

**总结：去掉Header或Header不合法,后台汇报 404 错误。带上合法的Header正常访问**





## 通过 Host 匹配



Host Route Predicate 接收一组参数，一组匹配的域名列表，这个模板是一个 ant 分隔的模板，用`.`号作为分隔符。它通过参数中的主机地址作为匹配规则。



~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Host=**.lison.com
 
 
~~~



![image-20240222181235735](typora-user-images/image-20240222181235735.png)

经测试以上两种 host 均可匹配到 host_route 路由，去掉 host 参数则会报 404 错误。



## 通过请求方式匹配

可以通过是 POST、GET、PUT、DELETE 等不同的请求方式来进行路由。





~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Method=GET
 
 
~~~

测试返回页面代码，证明匹配到路由，我们再以 POST 的方式请求测试。返回 404 没有找到，证明没有匹配上路由

## 通过请求路径匹配

Path Route Predicate 接收一个匹配路径的参数来判断是否走路由。





~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Path=/spring_service/{segment} # - Path=/spring_service/**
 
 
~~~

如果请求路径符合要求，则此路由将匹配，例如：/spring_service/1 或者 /spring_service/bar。



测试：

~~~
http://localhost:18003/spring_service/1
http://localhost:18003/spring_service/2
http://localhost:18003/spring_xxx/2
~~~



经过测试第一和第二条命令可以正常获取到页面返回值，最后一个命令报 404，证明路由是通过指定路由来匹配。

## 通过请求参数匹配

Query Route Predicate 支持传入两个参数，一个是属性名一个为属性值，属性值可以是正则表达式。

~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Query=token
 
 
~~~

这样配置，只要请求中包含 token 属性的参数即可匹配路由。



~~~
http://localhost:18003/spring_service/getServerProd?token=dfasdfas&id=xxx
~~~



经过测试发现只要请求汇总带有 smile 参数即会匹配路由，不带 token 参数则不会匹配。

还可以将 Query 的值以键值对的方式进行配置，这样在请求过来时会对属性值和正则进行匹配，匹配上才会走路由。

~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Query=token,lison.
 
 
~~~



这样只要当请求中包含 keep 属性并且参数值是以 lison 开头的长度为六位的字符串才会进行匹配和路由。

~~~
http://localhost:18003/spring_service/getServerProd?token=lison6&id=xxx
~~~

测试可以返回页面代码，将 token 的属性值改为 pubx 再次访问就会报 lison66, 证明路由需要匹配正则表达式才会进行路由。



## 通过请求 ip 地址进行匹配

Predicate 也支持通过设置某个 ip 区间号段的请求才会路由，RemoteAddr Route Predicate 接受 cidr 符号 (IPv4 或 IPv6) 字符串的列表(最小大小为 1)，例如 192.168.0.1/16 (其中 192.168.0.1 是 IP 地址，16 是子网掩码)。

~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - RemoteAddr=192.168.1.1/24
 
 
~~~



可以将此地址设置为本机的 ip 地址进行测试。

如果请求的远程地址是 192.168.1.10，则此路由将匹配。

## 自定义断言

### 如何自定义路由断言

我们可以查看内置的断言如何实现？举例：BetweenRoutePredicateFactory

~~~java

//断言工厂的类的命名规则为XXXXRoutePredicateFactory   需要 继承AbstractRoutePredicateFactory

public class BetweenRoutePredicateFactory extends AbstractRoutePredicateFactory<Config> {
    public static final String DATETIME1_KEY = "datetime1";
    public static final String DATETIME2_KEY = "datetime2";

    public BetweenRoutePredicateFactory() {
        super(Config.class);
    }

    public List<String> shortcutFieldOrder() {
        return Arrays.asList("datetime1", "datetime2"); //与配置文件映射
    }

  
  //判断逻辑方法
    public Predicate<ServerWebExchange> apply(final Config config) {
        Assert.isTrue(config.getDatetime1().isBefore(config.getDatetime2()), config.getDatetime1() + " must be before " + config.getDatetime2());
        return new GatewayPredicate() {
          //核心判断逻辑
            public boolean test(ServerWebExchange serverWebExchange) {
                ZonedDateTime now = ZonedDateTime.now();
                return now.isAfter(config.getDatetime1()) && now.isBefore(config.getDatetime2());
            }

            public String toString() {
                return String.format("Between: %s and %s", config.getDatetime1(), config.getDatetime2());
            }
        };
    }

  //参数配置类
    @Validated
    public static class Config {
        private @NotNull ZonedDateTime datetime1;
        private @NotNull ZonedDateTime datetime2;

        public Config() {
        }

        public ZonedDateTime getDatetime1() {
            return this.datetime1;
        }

        public Config setDatetime1(ZonedDateTime datetime1) {
            this.datetime1 = datetime1;
            return this;
        }

        public ZonedDateTime getDatetime2() {
            return this.datetime2;
        }

        public Config setDatetime2(ZonedDateTime datetime2) {
            this.datetime2 = datetime2;
            return this;
        }
    }
}
~~~

知道内置路由断言的实现细节，我们只需要按照它的实现方式，来按部就班的实现自己的路由断言即可。

### 实现

需求假设：年龄大于18岁，小于60岁的可以访问

1、先进行路由设置



~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Age=18,60 #年龄大于18岁，小于60岁的可以访问
 
 
~~~



2、自定义实现路由，根据前面的配置我们知道规则名为Age，那么我们的自定义的断言名称必须是AgeRoutePredicateFactory

~~~java
package com.lison.springcloudservice.config.predicates;

/**
 * @className: com.lison.springcloudservice.config.predicates-> AgeRoutePredicateFactory
 * @description:   自定义的断言工厂 1.名称必须是配置+RoutePredicateFactory 2.必须继承AbstractRoutePredicateFactory<配置类>
 * @author: Lison
 * @createDate: 2023-10-20
 */
import org.apache.commons.lang.StringUtils;
import org.springframework.cloud.gateway.handler.predicate.AbstractRoutePredicateFactory;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;

import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;


@Component
public class AgeRoutePredicateFactory extends AbstractRoutePredicateFactory<AgeRoutePredicateFactory.Config> {

    public AgeRoutePredicateFactory() {
        super(AgeRoutePredicateFactory.Config.class);
    }

    //读取配置文件的参数值，赋值到配置类中的属性上
    @Override
    public List<String> shortcutFieldOrder() {
        //顺序必须与yml文件中的配置顺序对应
        return Arrays.asList("minAge", "maxAge");
    }

    @Override
    public Predicate<ServerWebExchange> apply(AgeRoutePredicateFactory.Config config) {
        return new Predicate<ServerWebExchange>() {
            @Override
            public boolean test(ServerWebExchange serverWebExchange) {
                //serverWebExchange很强大，可以可以获取到很多内容
                String ageStr = serverWebExchange.getRequest().getQueryParams().getFirst("age");
                if (StringUtils.isNotEmpty(ageStr)) {
                    int age = Integer.parseInt(ageStr);
                    return age > config.getMinAge() && age < config.getMaxAge();
                }
                return false;
            }
        };
    }

    //用于接收参数
    public static class Config {
        private int minAge;
        private int maxAge;

        public int getMinAge() {
            return minAge;
        }

        public void setMinAge(int minAge) {
            this.minAge = minAge;
        }

        @Override
        public String toString() {
            return "Config{" +
                    "minAge=" + minAge +
                    ", maxAge=" + maxAge +
                    '}';
        }

        public int getMaxAge() {
            return maxAge;
        }

        public void setMaxAge(int maxAge) {
            this.maxAge = maxAge;
        }
    }
}


~~~

3、重启服务网关，测试自定义的路由断言，age=20可以访问，而age=15则无法访问







## 组合使用

上面为了演示各个 Predicate 的使用，我们是单个单个进行配置测试，其实可以将各种 Predicate 组合起来一起使用。



~~~yaml
spring:
  cloud:
    gateway:
      routes:
       - id: spring_service
        uri: http://localhost:18001
        predicates:
         - Host=**.lison.com
         - Path=/spring_service/** # 当请求路径满足Path指定的规则时,才进行路由转发
         - Cookie=token,lison
         - After=2023-10-20T08:30:00+08:00[Asia/Shanghai]
         - Before=2023-10-20T08:30:00+08:00[Asia/Shanghai]
         - Between=2023-10-20T08:30:00+08:00[Asia/Shanghai],2024-10-20T08:30:00+08:00[Asia/Shanghai]
         - Header=X-Request-Id, \d+
         - Host=**.lison.com
         - Method=GET,POST
         - Query=token,lison.
         - RemoteAddr=192.168.1.1/24
         - Age=18,60 
 
 
~~~

各种 Predicates 同时存在于同一个路由时，请求必须同时满足所有的条件才被这个路由匹配。

> 一个请求满足多个路由的谓词条件时，请求只会被首个成功匹配的路由转发







