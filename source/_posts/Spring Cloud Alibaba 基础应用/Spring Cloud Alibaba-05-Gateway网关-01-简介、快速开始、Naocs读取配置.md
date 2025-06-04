---
title: Spring Cloud Alibaba-05-Gateway网关-01-简介、快速开始、Naocs读取配置
date: 2025-04-05 19:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.10.16`

# Spring Cloud Alibaba-05-Gateway网关-01-简介、快速开始、Naocs读取配置

[toc]



## 网关简介

API Gateway（APIGW / API 网关），顾名思义，是出现在系统边界上的一个面向 API 的、串行集中式的强管控服务，这里的边界是企业 IT 系统的边界，可以理解为企业级应用防火墙，主要起到隔离外部访问与内部系统的作用。在微服务概念的流行之前，API 网关就已经诞生了，例如银行、证券等领域常见的前置机系统，它也是解决访问认证、报文转换、访问统计等问题的。

API 网关的流行，源于近几年来移动应用与企业间互联需求的兴起。移动应用、企业互联，使得后台服务支持的对象，从以前单一的Web应用，扩展到多种使用场景，且每种使用场景对后台服务的要求都不尽相同。这不仅增加了后台服务的响应量，还增加了后台服务的复杂性。随着微服务架构概念的提出，API网关成为了微服务架构的一个标配组件。

API 网关是一个服务器，是系统对外的唯一入口。API 网关封装了系统内部架构，为每个客户端提供定制的 API。所有的客户端和消费端都通过统一的网关接入微服务，在网关层处理所有非业务功能。API 网关并不是微服务场景中必须的组件，不管有没有 API 网关，后端微服务都可以通过 API 很好地支持客户端的访问。



在业界比较流行的网关，有下面这些:

**Ngnix+lua**

使用nginx的反向代理和负载均衡可实现对api服务器的负载均衡及高可用
lua是一种脚本语言,可以来编写一些简单的逻辑, nginx支持lua脚本

**Kong**

基于Nginx+Lua开发，性能高，稳定，有多个可用的插件(限流、鉴权等等)可以开箱即用。 问题: 只支持Http协议;二次开发，自由扩展困难;提供管理API，缺乏更易用的管控、配置方式。

**Zuul**

Netflix开源的网关，功能丰富，使用JAVA开发，易于二次开发 问题:缺乏管控，无法动态配 置;依赖组件较多;处理Http请求依赖的是Web容器，性能不如Nginx

**Spring Cloud Gateway**

Spring公司为了替换Zuul而开发的网关服务，将在下面具体介绍。

> SpringCloud alibaba技术栈中并没有提供自己的网关，我们可以采用Spring Cloud Gateway 来做网关



## Gateway简介

### 什么是spring cloud gateway

网关作为流量的入口，常用的功能包括路由转发、权限校验、限流等。

Spring Cloud Gateway 是基于 Spring 生态系统之上构建的 API 网关，包括：Spring 5，Spring Boot 2 和 Project Reactor。Spring Cloud Gateway 旨在提供一种简单而有效的方法来路由到 API，并为它们提供跨领域的关注点，例如：安全性，监视/指标，限流等。由于 Spring 5.0 支持 Netty，Http2，而 Spring Boot 2.0 支持 Spring 5.0，因此 Spring Cloud Gateway 支持 Netty 和 Http2 顺理成章。

客户端向 Spring Cloud Gateway 发出请求，如果请求与网关程序定义的路由匹配，则将其发送到网关 Web 处理程序，此处理程序运行特定的请求过滤器链。

过滤器之间用虚线分开的原因是过滤器可能会在发送代理请求之前或之后执行逻辑。所有 “pre” 过滤器逻辑先执行，然后执行代理请求，代理请求完成后，执行 “post” 过滤器逻辑。

**spring cloud gateway功能特性：**

（1）基于spring Framework5、Project Reactor和spring boot 2.0进行构建

（2）动态路由：能够匹配任何请求属性

（3）支持路径重写

（4）集成spring cloud服务发现功能（nacos）

（5）可集成流控级功能（sentinel）

（6）可以对路由指定易于编写的Predicate（断言）、Filter（过滤器）



**优点:**

性能强劲:是第一代网关Zuul的1.6倍

功能强大:内置了很多实用的功能，例如转发、监控、限流等

设计优雅，容易扩展

**缺点:**

其实现依赖Netty与WebFlux，不是传统的Servlet编程模型，学习成本高

不能将其部署在Tomcat、Jetty等Servlet容器里，只能打成jar包执行

需要Spring Boot 2.0及以上的版本，才支持

### 为什么要使用网关

单体应用：浏览器发起请求到单体应用所在的机器，应用从数据库查询数据原路返回给浏览器，对于单体应用来说是不需要网关的。
微服务：微服务的应用可能部署在不同机房，不同地区，不同域名下。此时客户端（浏览器/手机/软件工具）想要请求对应的服务，都需要知道机器的具体 IP 或者域名 URL，当微服务实例众多时，这是非常难以记忆的，对于客户端来说也太复杂难以维护。此时就有了网关，客户端相关的请求直接发送到网关，由网关根据请求标识解析判断出具体的微服务地址，再把请求转发到微服务实例。这其中的记忆功能就全部交由网关来操作了。

网关介于客户端与服务器之间的中间层，所有外部请求率先经过微服务网关，客户端只需要与网关交互，只需要知道网关地址即可。这样便简化了开发且有以下优点：

易于监控，可在微服务网关收集监控数据并将其推送到外部系统进行分析
易于认证，可在微服务网关上进行认证，然后再将请求转发到后端的微服务，从而无需在每个微服务中进行认证
减少了客户端与各个微服务之间的交互次数



### Gateway核心架构

 **基本概念**
路由(Route) 是 gateway 中最基本的组件之一，表示一个具体的路由信息载体。主要定义了下面的几个信息:

* **id**，路由标识符，区别于其他 Route。
* uri，路由指向的目的地 uri，即客户端请求最终被转发到的微服务。
* order，用于多个 Route 之间的排序，数值越小排序越靠前，匹配优先级越高。
* **predicate**，断言的作用是进行条件判断，只有断言都返回真，才会真正的执行路由。
* **filter**，过滤器用于修改请求和响应信息。



**执行流程**

![img](typora-user-images/u=641752398,1880469199&fm=253&fmt=auto&app=138&f=PNG.png)

**执行流程大体如下:**

1. Gateway Client向Gateway Server发送请求
2. 请求首先会被HttpWebHandlerAdapter进行提取组装成网关上下文
3. 然后网关的上下文会传递到DispatcherHandler，它负责将请求分发给 RoutePredicateHandlerMapping
4. RoutePredicateHandlerMapping负责路由查找，并根据路由断言判断路由是否可用
5. 如果过断言成功，由FilteringWebHandler创建过滤器链并调用
6. 请求会一次经过PreFilter–微服务–PostFilter的方法，最终返回响应



## Gateway快速开始



### 创建一个 spring-cloud-gateway的模块,导入相关依赖



~~~java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.lison</groupId>
        <artifactId>spring-cloud-alibaba-building</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <groupId>com.lison</groupId>
    <artifactId>spring-cloud-gateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-cloud-gateway</name>
    <description>spring-cloud-gateway</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>


    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
         </dependency>
    </dependencies>

</project>
~~~





### 创建主类

~~~java
@SpringBootApplication
public class SpringCloudGateWayApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudGateWayApplication.class, args);
    }

}
~~~



### 添加配置文件

~~~yaml
server:
  port: 18003
spring:
  application:
    name: spring-cloud-gateway
  cloud:
    gateway:
      routes: # 路由数组[路由 就是指定当请求满足什么条件的时候转到哪个微服务]
        - id: spring_service # 当前路由的标识, 要求唯一
          uri: http://localhost:18001 # 请求要转发到的地址
          order: 1 # 路由的优先级,数字越小级别越高
          predicates: # 断言(就是路由转发要满足的条件)
            - Path=/spring_service/** # 当请求路径满足Path指定的规则时,才进行路由转发
          filters: # 过滤器,请求在传递过程中可以通过过滤器对其进行一定的修改
            - StripPrefix=1 # 转发之前去掉1层路径


management:
  endpoint:
    web:
      exposure:
        include:'*'

~~~


访问 http://localhost:18003/spring_service/ 路由到 服务A http://localhost:18001/







### 启动项目, 并通过网关去访问微服务

http://localhost:18003/spring_service/gateway/test

![image-20240222152145318](typora-user-images/image-20240222152145318.png)

## Gateway访问进阶，Nacos 读取配置



现在在配置文件中写死了转发路径的地址, 前面我们已经分析过地址写死带来的问题, 接下来我们从 注册中心获取此地址。



### 加入Ncaos依赖

~~~java
<!--nacos客户端-->
 <dependency> 
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

~~~



### 主类上添加注解

~~~java
@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudGateWayApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudGateWayApplication.class, args);
    }

}
~~~



### 修改配置文件

~~~yaml
server:
  port: 18003
spring:
  application:
    name: spring-cloud-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
    gateway:
      discovery:
        locator:
          enabled: true # 让gateway可以发现nacos中的微服务
      routes: # 路由数组[路由 就是指定当请求满足什么条件的时候转到哪个微服务]
        - id: spring_service # 当前路由的标识, 要求唯一
          uri: lb://spring-cloud-service # lb指的是从nacos中按照名称获取微服务,并遵循负载均衡策略
          predicates: # 断言(就是路由转发要满足的条件)
            - Path=/spring_service/** # 当请求路径满足Path指定的规则时,才进行路由转发
          filters: # 过滤器,请求在传递过程中可以通过过滤器对其进行一定的修改
            - StripPrefix=1 # 转发之前去掉1层路径

~~~



### 测试



启动两个 service服务，配置不同端口

![image-20240222164404555](typora-user-images/image-20240222164404555.png)

![image-20240222164433038](typora-user-images/image-20240222164433038.png)

访问之前可以访问服务端口的服务：http://localhost:18003/spring_service/getServerProd

**访问到服务18001**

![image-20240222164533515](typora-user-images/image-20240222164533515.png)

**访问到服务18011**

![image-20240222164517003](typora-user-images/image-20240222164517003.png)
