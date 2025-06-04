---
title: Spring Cloud Alibaba-03-Nacos之配置中心、命名空间分组和DataID的关系
date: 2025-04-03 19:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.08.03`

# Spring Cloud Alibaba-03-Nacos之配置中心、命名空间分组和DataID的关系

[toc]





## 服务配置中心介绍

首先我们来看一下,微服务架构下关于配置文件的一些问题:

1. 配置文件相对分散。在一个微服务架构下，配置文件会随着微服务的增多变的越来越多，而且分散 在各个微服务中，不好统一配置和管理。
2. 配置文件无法区分环境。微服务项目可能会有多个环境，例如:测试环境、预发布环境、生产环 境。每一个环境所使用的配置理论上都是不同的，一旦需要修改，就需要我们去各个微服务下手动 维护，这比较困难。
3. 配置文件无法实时更新。我们修改了配置文件之后，必须重新启动微服务才能使配置生效，这对一 个正在运行的项目来说是非常不友好的。

基于上面这些问题，我们就需要配置中心的加入来解决这些问题。 配置中心的思路是:

* 首先把项目中各种配置全部都放到一个集中的地方进行统一管理，并提供一套标准的接口。
* 当各个服务需要获取配置的时候，就来配置中心的接口拉取自己的配置。当配置中心中的各种参数有更新的时候，也能通知到各个服务实时的过来同步最新的信息，使之动态更新。



常见的服务配置中心：

| 配置中心                             | 说明                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| **Apollo**                           | Apollo是由携程开源的分布式配置中心。特点有很多，比如:配置更新之后可以实时生效，支持灰 度发布功能，并且能对所有的配置进行版本管理、操作审计等功能，提供开放平台API。并且资料 也写的很详细。 |
| **Disconf**                          | Disconf是由百度开源的分布式配置中心。它是基于Zookeeper来实现配置变更后实时通知和生效 的。 |
| **SpringCloud Config              ** | 这是Spring Cloud中带的配置中心组件。它和Spring是无缝集成，使用起来非常方便，并且它的配 置存储支持Git。不过它没有可视化的操作界面，配置的生效也不是实时的，需要重启或去刷新。 |
| **Nacos**                            | 这是SpingCloud alibaba技术栈中的一个组件，前面我们已经使用它做过服务注册中心。其实它也 集成了服务配置的功能，我们可以直接使用它作为服务配置中心。 |



## Nacos Config基础配置

Nacos不仅仅可以作为注册中心来使用，同时它支持作为配置中心。就是将nacos当做一个服务端，将各个微服务看成是客户端，我们
将各个微服务的配置文件统一存放在nacos上，然后各个微服务从nacos上拉取配置即可



1、 搭建nacos环境【使用现有的nacos环境即可】
2 、在微服务中引入nacos的依赖



~~~xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>

~~~

3、在微服务中添加nacos config的配置

> 注意:不能使用原来的application.yml作为配置文件，而是新建一个bootstrap.yml作为配置文件

配置文件优先级(由高到低):

> bootstrap.properties -> bootstrap.yml -> application.properties -> application.yml



~~~yaml
spring:
  application:
    name: spring-cloud-service

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        file-extension: yaml # 配置文件格式 
        
  profiles: 
	  active: dev # 环境标识      
~~~



在nacos中添加配置 点击配置列表，点击右边+号，新建配置。在新建配置过程中，要注意下面的细节:

>1)Data ID不能随便写，要跟配置文件中的对应，对应关系如图所示
>2)配置文件格式要跟配置文件的格式对应，且目前仅仅支持YAML和Properties
>3)配置内容按照上面选定的格式书写

![image-20231008103734081](typora-user-images/image-20231008103734081.png)

注释本地的application.yam中的内容， 启动程序进行测试

如果依旧可以成功访问程序，说明我们nacos的配置中心功能已经实现



## Nacos Config深入



### 配置动态刷新

在基础配置中，但是此时如果修改了配置，我们的程序是无法读取到的，因此，我们需要开启配置的动态刷新功能。



在nacos中的 spring-cloud-service-dev.yaml配置项中添加下面配置:

~~~yaml
config:
    appName: service
~~~





**方式一**: 硬编码方式

~~~java
@RestController
public class TestController {
    @Autowired
    private ConfigurableApplicationContext applicationContext;
  
  
	 @GetMapping(value = "/getConfigName")
    public String getConfigName(){
        return applicationContext.getEnvironment().getProperty("config.appName");
    }
}
~~~



**方式二**: 注解方式(推荐)

~~~java

@RestController
@RefreshScope//只需要在需要动态读取配置的类上添加此注解就可以
public class TestController {

    @Value("${config.appName}")
    private String configName;
    
	//2 注解方式 
	   @GetMapping(value = "/getConfigNames")
    public String getConfigNames(){
        return configName;
    }
}
~~~





### 配置共享

当配置越来越多的时候，我们就发现有很多配置是重复的，这时候就考虑可不可以将公共配置文件 提取出来，然后实现共享呢?当然是可以的。接下来我们就来探讨如何实现这一功能。

**同一个微服务的不同环境之间共享配置**

如果想在同一个微服务的不同环境之间实现配置共享，其实很简单。
只需要提取一个以 spring.application.name 命名的配置文件，然后将其所有环境的公共配置放在里 面即可。



1、新建一个名为spring-cloud-service.yaml配置存放服务的公共配置

![image-20231024095726856](typora-user-images/image-20231024095726856.png)

2 新建一个名为spring-cloud-service-test.yaml配置存放测试环境的配置

![image-20231024095816182](typora-user-images/image-20231024095816182.png)

3、 新建一个名为spring-cloud-service-dev.yaml配置存放测试环境的配置![image-20231024095843156](typora-user-images/image-20231024095843156.png)







~~~java

@RestController
@RefreshScope//只需要在需要动态读取配置的类上添加此注解就可以
public class TestController {

    @Value("${config.appName}")
    private String configName;
    @Value("${config.env}")
    private String env;


    @GetMapping(value = "/getConfigNames")
    public String getConfigNames(){
        return configName+"环境"+env;
    }
~~~





4、访问测试



![image-20231024094558547](typora-user-images/image-20231024094558547.png)

接下来，将active设置成test，再次访问，观察结果

~~~yaml
spring:
  profiles:
     active: test # 环境标识

~~~





### **不同微服务中间共享配置**

不同为服务之间实现配置共享的原理类似于文件引入，就是定义一个公共配置，然后在当前配置中引
入。



~~~yaml
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://spring_cloud_service?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
    username: root
    password: root
  jpa:
    properties:
      hibernate:
        hbm2ddl:
          auto: update
        dialect: org.hibernate.dialect.MySQL5InnoDBDialect
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848


~~~



修改bootstrap.yaml

~~~yam
spring:
  application:
    name: service-product
  cloud:
    nacos:
      config:
		server-addr: 127.0.0.1:8848 #nacos中心地址
		file-extension: yaml # 配置文件格式
		shared-dataids: spring-cloud-all.yaml # 配置要引入的配置 refreshable-dataids: spring-cloud-all.yaml # 配置要实现动态配置刷新的配置 
  profiles:
	active: dev # 环境标识 

~~~



启动微服务进行测试

## Nacos的几个概念



**命名空间(Namespace)**

命名空间可用于进行不同环境的配置隔离。一般一个环境划分到一个命名空间

**配置分组(Group)**

配置分组用于将不同的服务可以归类到同一分组。一般将一个项目的配置分到一组

**配置集(Data ID)**

在系统中，一个配置文件通常就是一个配置集。一般微服务的配置就是一个配置集

![image-20231024101307177](typora-user-images/image-20231024101307177.png)
