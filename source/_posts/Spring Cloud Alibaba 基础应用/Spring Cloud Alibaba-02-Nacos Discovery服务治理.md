---
title: Spring Cloud Alibaba-02-Nacos Discovery服务治理及负载均衡
date: 2025-04-02 19:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.05.10`

# Spring Cloud Alibaba-02-Nacos Discovery服务治理及负载均衡

[toc]



## 服务治理介绍



先来思考一个问题

通过上一章的操作，我们已经可以实现微服务之间的调用。但是我们把服务提供者的网络地址 (ip，端口)等硬编码到了代码中，这种做法存在许多问题:

一旦服务提供者地址变化，就需要手工修改代码

一旦是多个服务提供者，无法实现负载均衡功能

一旦服务变得越来越多，人工维护调用关系困难

那么应该怎么解决呢， 这时候就需要通过注册中心动态的实现服务治理。

### 什么是服务治理

服务治理是微服务架构中最核心最基本的模块。用于实现各个微服务的自动化注册与发现。

服务注册:在服务治理框架中，都会构建一个注册中心，每个服务单元向注册中心登记自己提供服务的详细信息。并在注册中心形成一张服务的清单，服务注册中心需要以心跳的方式去监测清单中的服务是否可用，如果不可用，需要在服务清单中剔除不可用的服务。

服务发现:服务调用方向服务注册中心咨询服务，并获取所有服务的实例清单，实现对具体服务实例的访问

![image-20230923151411555](typora-user-images/image-20230923151411555.png)



通过上面的调用图会发现，除了微服务，还有一个组件是**服务注册中心**，它是微服务架构非常重要 的一个组件，在微服务架构里主要起到了协调者的一个作用。注册中心一般包含如下几个功能:

1. 服务发现:
   服务注册:保存服务提供者和服务调用者的信息
   服务订阅:服务调用者订阅服务提供者的信息，注册中心向订阅者推送提供者的信息
2. 服务配置:
   配置订阅:服务提供者和服务调用者订阅微服务相关的配置
   配置下发:主动将配置推送给服务提供者和服务调用者
3. 服务健康检测：
   检测服务提供者的健康情况，如果发现异常，执行服务剔除

### 常见的注册中心

**Zookeeper**

zookeeper是一个分布式服务框架，是Apache Hadoop 的一个子项目，它主要是用来解决分布式 应用中经常遇到的一些数据管理问题，如:统一命名服务、状态同步服务、集群管理、分布式应用 配置项的管理等。

**Eureka**

Eureka是Springcloud Netflix中的重要组件，主要作用就是做服务注册和发现。但是现在已经闭 源

**Consul**

Consul是基于GO语言开发的开源工具，主要面向分布式，服务化的系统提供服务注册、服务发现 和配置管理的功能。Consul的功能都很实用，其中包括:服务注册/发现、健康检查、Key/Value 存储、多数据中心和分布式一致性保证等特性。Consul本身只是一个二进制的可执行文件，所以 安装和部署都非常简单，只需要从官网下载后，在执行对应的启动脚本即可。

**Nacos**

Nacos是一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。它是 Spring Cloud Alibaba 组件之一，负责服务注册发现和服务配置，可以这样认为nacos=eureka+config。





## Nacos介绍



* Nacos是SpringCloudAlibaba架构中最重要的组件。
* Nacos 是一个更易于帮助构建云原生应用的动态服务发现、配置和服务管理平台，提供注册中心、配置中心和动态 DNS 服务三大功能。能够无缝对接Springcloud、Spring、Dubbo等流行框架。
* nacos和eureka功能对比

| 功能模块     | Nacos | Eureka | 功能说明                                                     |
| ------------ | ----- | ------ | ------------------------------------------------------------ |
| 注册中心     | √     | √      | 服务治理，服务中心化注册                                     |
| 配置中心     | √     | ×      | eureka需要配合springcloud config实现                         |
| 配置动态刷新 | √     | ×      | nacos通过netty保持tcp长链接进行推送，eureka需要配合mq实现配置动态 |
| 可用区az     | √     | √      | 对服务集群划分不同区域，实现区域隔离，并提供灾难级自动切换   |
| 分组         | √     | ×      | nacos根据不同的业务、环境进行分组管理（namespace,group       |
| 元数据       | √     | √      | 提供服务标签数据（环境、服务标识）                           |
| 权重         | √     | ×      | nacos提供权重设置，调整承载流量压力                          |
| 健康检查     | √     | √      | nacos提供服务端或者客户端发起的健康监测，eureka是有客户端发起心跳 |
| 负载均衡     | √     | √      | 均提供负载均衡策略，eureka采用ribbon                         |

* nacos支持a（高可用）p（分区容错）和c（一致性）p的切换默认为ap, eureka仅支持ap，zookeeper仅支持c



## nacos能做什么？



* 服务注册发现和服务健康监测：Nacos支持基于DNS和基于RPC的服务发现，服务端可以通过SDK或者Api进行服务注册，相应的服务消费者可以使用DNS或者Http查找的方式获取服务列表。Nacos同时提供对服务的实时健康检查，阻止想不健康的主机或服务发送请求，与Eureka类似Nacos也有友好的控制台界面。
* 动态DNS服务：支持权重路由，更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。
* 动态配置服务：接触过SpringCloud应该对config有所了解，那么配置中心也就很好理解，Nacos支持动态的配置管理，将服务的配置信息分环境分类别外部管理，并且支持热更新。不过与Config不同Nacos的配置信息存储与数据库中，支持配置信息的监听和版本回滚。
* 服务及元数据管理：Nacos 能让您从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA （服务等级协议）以及最首要的 metrics 统计数据（默认不开启暴露需要修改配置）。可以搭建搭建prometheus采集Nacos metrics数据也可以搭建搭建grafana图形化展示metrics数据





## Nacos下载安装

**官网网址：**https://nacos.io/zh-cn/index.html

![image-20230921165730484](typora-user-images/image-20230921165730484.png)



**官网文档网址**：https://nacos.io/zh-cn/docs/quick-start.html

**注意：**使用官网推荐的稳定版本：下载地址：https://github.com/alibaba/nacos/releases



![image-20230921170222187](typora-user-images/image-20230921170222187.png)

### **执行命令**



**Linux/Unix/Mac**

启动命令(standalone代表着单机模式运行，非集群模式):

```
sh startup.sh -m standalone
```



**Windows**

启动命令(standalone代表着单机模式运行，非集群模式):

```
startup.cmd -m standalone
```



### **执行结果**



![image-20230921171251668](typora-user-images/image-20230921171251668.png)







### 验证

得到结果以后为了验证是否成功开启Nacos，我们需要访问：http://localhost:8848/nacos

![image-20230921170734063](typora-user-images/image-20230921170734063.png)

出现此界面表示已经成功启动Nacos，默认的账号密码是：nacos/nacos

![image-20230921171005578](typora-user-images/image-20230921171005578.png)





## 引入Nacos Discovery进行服务注册/发现

服务发现是微服务架构中的关键组件之一。在这样的架构中，手动为每个客户端配置服务列表可能是一项艰巨的任务，并且使得动态扩展极其困难。Nacos Discovery 帮助您自动将您的服务注册到 Nacos 服务器，Nacos 服务器会跟踪服务并动态刷新服务列表。此外，Nacos Discovery 将服务实例的一些元数据，如主机、端口、健康检查 URL、主页等注册到 Nacos。

 学习任何知识我们都需要从它的官方文档入手，所以我们直接来看官网给我们提供的文档：https://spring.io/projects/spring-cloud-alibaba#learn



### 创建新项目

聚合项目：由于聚合带来的诸多好处，在SpringBoot项目开发中也广泛采用，开发中将SpringBoot项目按照功能分成子模块开发，所以我们在使用Spring Cloud Alibaba完成项目的时候，也是采用聚合项目来完成。

**父项目pom**



~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>


    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.13.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>com.lison</groupId>
    <artifactId>spring-cloud-alibaba-building</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>


    <properties>
        <java.version>17</java.version>
        <lison.project.version>1.0.0-SNAPSHOT</lison.project.version>
        <maven.plugin.version>3.8.1</maven.plugin.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-cloud-alibaba-version>2.2.5.RELEASE</spring-cloud-alibaba-version>
    </properties>

    <modules>
        <module>spring-boot-building</module>
    </modules>


    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba-version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

</project>

~~~



**子项目pom**

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.lison</groupId>
        <artifactId>spring-cloud-alibaba-building</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.lison</groupId>
    <artifactId>spring-boot-building</artifactId>
    <version>${lison.project.version}</version>
    <name>${project.artifactId}</name>
    <packaging>jar</packaging>
    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
    </dependencies>

</project>

~~~

**子项目yml**



~~~yaml
server:
  port: 18000
spring:
  application:
    name: spring-boot-building

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

management:
  endpoint:
    web:
      exposure:
        include:'*'

~~~



**启动类**

~~~java
package com.lison.springbootbuilding;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class SpringBootBuildingApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootBuildingApplication.class, args);
    }

}

~~~



### 测试

**注意，在启动项目前一定要启动Nacos**

![image-20230923154459874](typora-user-images/image-20230923154459874.png)









## Nacos服务消费者和负载均衡



### 什么是负载均衡

通俗的讲， 负载均衡就是将负载(工作任务，访问请求)进行分摊到多个操作单元(服务器,组件)上进行执行。
根据负载均衡发生位置的不同,一般分为服务端负载均衡和客户端负载均衡。 服务端负载均衡指的是发生在服务提供者一方,比如常见的nginx负载均衡而客户端负载均衡指的是发生在服务请求的一方，也就是在发送请求之前已经选好了由哪个实例处理请求。



![image-20230923152344429](typora-user-images/image-20230923152344429.png)

我们在微服务调用关系中一般会选择客户端负载均衡，也就是在服务调用的一方来决定服务由哪个提供者执行



### 负载均衡

创建一个spring-cloud-service 服务

 pom.xml配置



~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.lison</groupId>
        <artifactId>spring-cloud-alibaba-building</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.lison</groupId>
    <artifactId>spring-cloud-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-cloud-service</name>
    <description>spring-cloud-service</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>


    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
    </dependencies>



    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-antrun-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>run</goal>
                        </goals>
                        <configuration>
                            <tasks>
                                <!--suppress UnresolvedMavenProperty -->
                                <copy overwrite="true"
                                      tofile="${session.executionRootDirectory}/target/${project.artifactId}.jar"
                                      file="${project.build.directory}/${project.artifactId}.jar" />
                            </tasks>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

~~~

**yml配置**

~~~java
server:
  port: 18001
spring:
  application:
    name: spring-cloud-service

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

management:
  endpoint:
    web:
      exposure:
        include:'*'

~~~



**启动类**

~~~java
package com.lison.springcloudservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudServiceApplication.class, args);
    }

}

~~~



接下来开始修改 spring-cloud-service模块的代码， 将其注册到nacos服务上

![image-20230923154644281](typora-user-images/image-20230923154644281.png)

#### 基于Ribbon实现负载均衡

Ribbon是Spring Cloud的一个组件， 它可以让我们使用一个注解就能轻松的搞定负载均衡

1、在RestTemplate 的生成方法上添加@LoadBalanced注解

~~~java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

~~~



2、修改服务调用的方法

~~~java
@RestController
public class TestController {
    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/naocs/consumer")
    public String getServerPort(){

        return restTemplate.getForObject( "http://spring-cloud-service/getServerProd",String.class);

    }
}
~~~





3、通过idea再启动一个 spring-cloud-service 微服务，设置其端口为18011

![image-20230923162506692](typora-user-images/image-20230923162506692.png)



4、通过nacos查看微服务的启动情况



![image-20230923162555344](typora-user-images/image-20230923162555344.png)





Ribbon支持的负载均衡策略

Ribbon内置了多种负载均衡策略,内部负载均衡的顶级接口为 com.netflix.loadbalancer.IRule , 具体的负载策略如下图所示:



| 策略名                    | 策略描述                                                     | **实现说明**                                                 |
| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| BestAvailableRule         | 选择一个最小的并发 请求的server                              | 逐个考察Server，如果Server被 tripped了，则忽略，在选择其中 ActiveRequestsCount最小的serve |
| AvailabilityFilteringRule | 过滤掉那些因为一直 连接失败的被标记为 circuit tripped的后 端server，并过滤掉 那些高并发的的后端 server(active connections 超过配 置的阈值) | 使用一个AvailabilityPredicate来包含 过滤server的逻辑，其实就就是检查 status里记录的各个server的运行状态 |
| WeightedResponseTimeRule  | 根据相应时间分配一 个weight，相应时 间越长，weight越 小，被选中的可能性 越低。 | 一个后台线程定期的从status里面读 取评价响应时间，为每个server计算 一个weight。Weight的计算也比较简 单responsetime 减去每个server自己 平均的responsetime是server的权 重。当刚开始运行，没有形成statas 时，使用roubine策略选择server。 |
| RetryRule                 | 对选定的负载均衡策略机上重试机制。                           | 在一个配置时间段内当选择server不 成功，则一直尝试使用subRule的方 式选择一个可用的server |
| RoundRobinRule            | 轮询方式轮询选择 server                                      | 轮询index，选择index对应位置的 server                        |
| RandomRule                | 随机选择一个server                                           | 在index上随机，选择index对应位置 的server                    |
| ZoneAvoidanceRule         | 复合判断server所在 区域的性能和server 的可用性选择server     | 使用ZoneAvoidancePredicate和 AvailabilityPredicate来判断是否选择 某个server，前一个判断判定一个 zone的运行性能是否可用，剔除不可 用的zone(的所有server)， AvailabilityPredicate用于过滤掉连接 数过多的Server。 |

我们可以通过修改配置来调整Ribbon的负载均衡策略，具体代码如下：

~~~yaml
spring-cloud-service: # 调用的提供者的名称 
	ribbon: 
   		NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule


~~~





### 基于Feign实现服务调用

#### 什么是Feign

Feign是Spring Cloud提供的一个声明式的伪Http客户端， 它使得调用远程服务就像调用本地服务 一样简单， 只需要创建一个接口并添加一个注解即可。

Nacos很好的兼容了Feign， Feign默认集成了 Ribbon， 所以在Nacos下使用Fegin默认就实现了负 载均衡的效果。

#### Feign的使用

1、加入Feign依赖

~~~xml
<!--fegin组件--> 
<dependency> 
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

~~~



2、在主类上添加Fegin的注解

~~~java

@SpringBootApplication 
@EnableDiscoveryClient 
@EnableFeignClients
//开启Fegin
~~~





3创建一个service， 并使用Fegin实现微服务调用



~~~java
@FeignClient("spring-cloud-service")
public interface ITestService {
    //指定调用提供者的哪个方法
    @GetMapping(value = "/getServerProd")
    String getServerPort();
}
~~~



4、修改controller代码，并启动验证



~~~java
@RestController
public class TestController {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private ITestService iTestService;

    @GetMapping(value = "/naocs/consumer")
    public String getServerPort(){

        return iTestService.getServerPort();

    }
}

~~~



5、重启buding微服务,访问：http://127.0.0.1:18000/naocs/consumer 查看效果



