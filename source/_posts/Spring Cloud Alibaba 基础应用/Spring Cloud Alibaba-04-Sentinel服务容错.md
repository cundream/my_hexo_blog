---
title: Spring Cloud Alibaba-04-Sentinel服务容错
date: 2025-04-04 19:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.09.10`

# Spring Cloud Alibaba-04-Sentinel服务容错

[toc]



## 高并发带来的问题

在微服务架构中，我们将业务拆分成一个个的服务，服务与服务之间可以相互调用，但是由于网络 原因或者自身的原因，服务并不能保证服务的100%可用，如果单个服务出现问题，调用这个服务就会 出现网络延迟，此时若有大量的网络涌入，会形成任务堆积，最终导致服务瘫痪。



## 服务雪崩效应

在分布式系统中,由于网络原因或自身的原因,服务一般无法保证 100% 可用。如果一个服务出现了 问题，调用这个服务就会出现线程阻塞的情况，此时若有大量的请求涌入，就会出现多条线程阻塞等 待，进而导致服务瘫痪。

由于服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是 服务故障的 “雪崩效应” 。

![image-20231024102201145](typora-user-images/image-20231024102201145.png)



雪崩发生的原因多种多样，有不合理的容量设计，或者是高并发下某一个方法响应变慢，亦或是某台机器的资源耗尽。我们无法完全杜绝雪崩源头的发生，只有做好足够的容错，保证在一个服务发生问题，不会影响到其它服务的正常运行。也就是"雪落而不雪崩"。





## 常见容错方案

要防止雪崩的扩散，我们就要做好服务的容错，容错说白了就是保护自己不被猪队友拖垮的一些措 施, 下面介绍常见的服务容错思路和组件。

**常见的容错思路**

常见的容错思路有隔离、超时、限流、熔断、降级这几种，下面分别介绍一下。

- **隔离**

  它是指将系统按照一定的原则划分为若干个服务模块，各个模块之间相对独立，无强依赖。当有故
  障发生时，能将问题和影响隔离在某个模块内部，而不扩散风险，不波及其它模块，不影响整体的
  系统服务。常见的隔离方式有:线程池隔离和信号量隔离.



![image-20231024104338016](typora-user-images/image-20231024104338016.png)



* **超时**

在上游服务调用下游服务的时候，设置一个最大响应时间，如果超过这个时间，下游未作出反应，就断开请求，释放掉线程。

![image-20231201143940420](typora-user-images/image-20231201143940420.png)



* **限流**

限流就是限制系统的输入和输出流量已达到保护系统的目的。为了保证系统的稳固运行,一旦达到 的需要限制的阈值,就需要限制流量并采取少量措施以完成限制流量的目的。

![image-20231201144020748](typora-user-images/image-20231201144020748.png)

* **熔断**

在互联网系统中，当下游服务因访问压力过大而响应变慢或失败，上游服务为了保护系统整 体的可用性，可以暂时切断对下游服务的调用。这种牺牲局部，保全整体的措施就叫做熔断。

![image-20231201144119343](typora-user-images/image-20231201144119343.png)

服务熔断一般有三种状态:

* 熔断关闭状态(Closed)：  服务没有故障时，熔断器所处的状态，对调用方的调用不做任何限制 
* 熔断开启状态（Open）：  后续对该服务接口的调用不再经过网络，直接执行本地的fallback方法 
* 半熔断状态（Half-Open）	：  尝试恢复服务调用，允许有限的流量调用该服务，并监控调用成功率。如果成功率达到预期，则说明服务已恢复，进入熔断关闭状态;如果成功率仍旧很低，则重新进入熔断关闭状态。





* **降级**

降级其实就是为服务提供一个托底方案，一旦服务无法正常调用，就使用托底方案。

![image-20231201153117052](typora-user-images/image-20231201153117052.png)



**常见的容错组件**

**Hystrix**
Hystrix是由Netflix开源的一个延迟和容错库，用于隔离访问远程系统、服务或者第三方库，防止 级联失败，从而提升系统的可用性与容错性。

**Resilience4J**
Resilicence4J一款非常轻量、简单，并且文档非常清晰、丰富的熔断工具，这也是Hystrix官方推 荐的替代产品。不仅如此，Resilicence4j还原生支持Spring Boot 1.x/2.x，而且监控也支持和 prometheus等多款主流产品进行整合。

**Sentinel**
Sentinel 是阿里巴巴开源的一款断路器实现，本身在阿里内部已经被大规模采用，非常稳定。 下面是三个组件在各方面的对比:

|                | Sentinel                                                   | Hystrix                | Resilience4J                     |
| -------------- | ---------------------------------------------------------- | ---------------------- | -------------------------------- |
| 隔离策略       | 信号量隔离（并发线程数限流）                               | 线程池隔离/信号量隔离  | 信号量隔离                       |
| 熔断降级策略   | 基于响应时间、异常比率、异常数                             | 基于异常比率           | 基于异常比率、响应时间           |
| 实时统计实现   | 活动窗口（LeapArray）                                      | 滑动窗口（基于RxJava） | Ring Bit Buffer                  |
| 动态规则配置   | 支持多数据源                                               | 支持多数据源           | 有限支持                         |
| 扩展性         | 多个扩展点                                                 | 插件的形式             | 插件的形式                       |
| 基于注解的支持 | 支持                                                       | 支持                   | 支持                             |
| 限流           | 基于QPS,支持基于调用关系的限流                             | 不支持                 | Rate Limiter                     |
| 流量整形       | 支持预热模式、匀速器模式、预热排队模式                     | 不支持                 | 简单的Rate Limiter模式           |
| 系统自适应保护 | 支持                                                       | 不支持                 | 不支持                           |
| 控制台         | 提供开箱即用的控制台，可配置规则、查看秒级监控、机器发现等 | 简单的监控查 看        | 不提供控制台，可对接其它监控系统 |



## Sentinel入门

### 什么是Sentinel

Sentinel (分布式系统的流量防卫兵) 是阿里开源的一套用于**服务容错**的综合性解决方案。它以流量 为切入点, 从**流量控制、熔断降级、系统负载保护**等多个维度来保护服务的稳定性。



Sentinel具有以下特征:

Sentinel 具有以下特征:

**丰富的应用场景 :**Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景, 例如秒杀(即 突发流量控制在系统容量可以承受的范围)、消息削峰填谷、集群流量控制、实时熔断下游不可用 应用等。
完备的实时监控:Sentinel 提供了实时的监控功能。通过控制台可以看到接入应用的单台机器秒 级数据, 甚至 500 台以下规模的集群的汇总运行情况。
**广泛的开源生态**:Sentinel 提供开箱即用的与其它开源框架/库的整合模块, 例如与 Spring Cloud、Dubbo、gRPC 的整合。只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。

**完善的 SPI 扩展点:**Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快 速地定制逻辑。例如定制规则管理、适配动态数据源等。

**Sentinel 分为两个部分:**

* 核心库(Java 客户端)不依赖任何框架/库,能够运行于所有 Java 运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。
* 控制台(Dashboard)基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等 应用容器。



### 微服务集成Sentinel

为微服务集成Sentinel非常简单, 只需要加入Sentinel的依赖即可

**1、在pom.xml中加入下面依赖**

~~~xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

~~~



**2、application.yml 配置sentinel**
**`注意：yml配置client-ip 是本地ip才行`**

~~~yaml
spring:
    cloud:
        sentinel:
            transport:
                dashboard: 192.168.32.131:8858
                client-ip: 192.168.32.1
                port: 8719

~~~







**3、编写一个Controller测试使用**

~~~java
@RestController
public class TestSentinelController {

    @RequestMapping("/sentinel/message1")
    public String message1() {
        return "message1";
    }
    @RequestMapping("/sentinel/message2")
    public String message2() {
        return "message2";
    }
    
}
~~~



### 安装Sentinel控制台

Sentinel 提供一个轻量级的控制台, 它提供机器发现、单机资源实时监控以及规则管理等功能。

**1、jar包方式安装**

~~~
下载jar包,解压到文件夹 https://github.com/alibaba/Sentinel/releases 


启动：
# 直接使用jar命令启动项目(控制台本身是一个SpringBoot项目)
java -Dserver.port=8858 -Dcsp.sentinel.dashboard.server=localhost:8858 - Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.1.jar 


~~~



2、docker方式安装

~~~

//拉去sentinel镜像
docker pull bladex/sentinel-dashboard:1.8.1



//启动容器
docker run --name sentinel  -d -p 8858:8858 -p 8719:8719 -d bladex/sentinel-dashboard:1.8.1 -e username=sentinel -e password=sentinel -e server=localhost:8858
~~~



通过浏览器访问localhost:8858 进入控制台 ( 默认用户名密码是 sentinel/sentinel )





注意：重启后还是看不到自己注册接口服务？

**原因：Sentinel是 懒加载机制所以呢，需要访问一下接口即可再去访问Sentinel 就有数据了**





**补充:了解控制台的使用原理**

Sentinel的控制台其实就是一个SpringBoot编写的程序。我们需要将我们的微服务程序注册到控制台上, 即在微服务中指定控制台的地址, 并且还要开启一个跟控制台传递数据的端口, 控制台也可以通过此端口 调用微服务中的监控程序获取微服务的各种信息。

![image-20240219114233026](typora-user-images/image-20240219114233026.png)



![image-20240219113719157](typora-user-images/image-20240219113719157.png)







## 实现一个接口的限流



1、通过控制台为message1添加一个流控规则



![image-20240219114343151](typora-user-images/image-20240219114343151.png)



2、通过控制台快速频繁访问, 观察效果

![image-20240219114515952](typora-user-images/image-20240219114515952.png)

## Sentinel的概念和功能



### 基本概念

* **资源**

资源就是Sentinel要保护的东西

资源是Sentinel的关键概念，它可以是Java应用程序中的任何内容，可以是一个服务，也可以是一个方法，甚至可以是一段代码

>上述案例中message1方法就可以认为是一个资源

* **规则**

规则就是用来定义如何保护资源的

作为资源之上，定义以什么样的方式保护资源，主要包括流量控制规则、熔断降级规则以及系统保护规则。

>上述案例中message1资源设置一种流控规则，限制了进入message1的流量

### 重要功能



![image-20240219144606520](typora-user-images/image-20240219144606520.png)

Sentinel的主要功能就是容错,主要体现为下面这三个:

**1、流量控制**
流量控制在网络传输中是一个常用的概念，它用于调整网络包的数据。任意时间到来的请求往往是 随机不可控的，而系统的处理能力是有限的。我们需要根据系统的处理能力对流量进行控制。 Sentinel 作为一个调配器，可以根据需要把随机的请求调整成合适的形状。

**2、熔断降级**

当检测到调用链路中某个资源出现不稳定的表现，例如请求响应时间长或异常比例升高的时候，则
对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联故障。



**Sentinel 对这个问题采取了两种手段:**

(1)通过并发线程数进行限制

Sentinel 通过限制资源并发线程的数量，来减少不稳定资源对其它资源的影响。当某个资源 出现不稳定的情况下，例如响应时间变长，对资源的直接影响就是会造成线程数的逐步堆 积。当线程数在特定资源上堆积到一定的数量之后，对该资源的新请求就会被拒绝。堆积的 线程完成任务后才开始继续接收请求。

(2)通过响应时间对资源进行降级

除了对并发线程数进行控制以外，Sentinel 还可以通过响应时间来快速降级不稳定的资源。 当依赖的资源出现响应时间过长后，所有对该资源的访问都会被直接拒绝，直到过了指定的 时间窗口之后才重新恢复。



**Sentinel 和 Hystrix 的区别**

>两者的原则是一致的, 都是当一个资源出现问题时, 让其快速失败, 不要波及到其它服务 但是在限制的手段上, 确采取了完全不一样的方法:
>
>~~~
>       Hystrix 采用的是线程池隔离的方式, 优点是做到了资源之间的隔离, 缺点是增加了线程 切换的成本。 
>
>       Sentinel 采用的是通过并发线程的数量和响应时间来对资源做限制。 
>~~~



**3、系统负载保护**

Sentinel 同时提供系统维度的自适应保护能力。当系统负载较高的时候，如果还持续让 请求进入可能会导致系统崩溃，无法响应。在集群环境下，会把本应这台机器承载的流量转发到其 它的机器上去。如果这个时候其它的机器也处在一个边缘状态的时候，Sentinel 提供了对应的保 护机制，让系统的入口流量和系统的负载达到一个平衡，保证系统在能力范围之内处理最多的请 求。总之一句话: 我们需要做的事情，就是在Sentinel的资源上配置各种各样的规则，来实现各种容错的功 能。



## Sentinel规则



### 流程控制

流量控制，其原理是监控应用流量的QPS(每秒查询率) 或并发线程数等指标，当达到指定的阈值时 对流量进行控制，以避免被瞬时的流量高峰冲垮，从而保障应用的高可用性。

第1步: 点击簇点链路，我们就可以看到访问过的接口地址，然后点击对应的流控按钮，进入流控规则配 置页面。新增流控规则界面如下:

![image-20240219150634816](typora-user-images/image-20240219150634816.png)



**资源名:**唯一名称，默认是请求路径，可自定义

针对来源:指定对哪个微服务进行限流，默认指default，意思是不区分来源，全部限制

**阈值类型/单机阈值:**

* QPS(每秒请求数量): 当调用该接口的QPS达到阈值的时候，进行限流
* 线程数:当调用该接口的线程数达到阈值的时候，进行限流

**是否集群:**暂不需要集群 接下来我们以QPS为例来研究限流规则的配置。



#### 简单配置

我们先做一个简单配置，设置阈值类型为QPS，单机阈值为3。即每秒请求量大于3的时候开始限流。

接下来，在流控规则页面就可以看到这个配置

![image-20240219150821657](typora-user-images/image-20240219150821657.png)

然后快速访问 /order/message1 接口，观察效果。此时发现，当QPS > 3的时候，服务就不能正常响 应，而是返回Blocked by Sentinel (flow limiting)结果。

![image-20240219150910898](typora-user-images/image-20240219150910898.png)

#### 配置流控模式

![image-20240219151031448](typora-user-images/image-20240219151031448.png)

sentinel共有三种流控模式，分别是:

**直接**(默认):接口达到限流条件时，开启限流

**关联**:当关联的资源达到限流条件时，开启限流 [适合做应用让步]

**链路**:当从某个接口过来的资源达到限流条件时，开启限流



下面呢分别演示三种模式:

**直接流控模式**

直接流控模式是最简单的模式，当指定的接口达到限流条件时开启限流。上面案例使用的就是直接流控模式。

**关联流控模式**

关联流控模式指的是，当指定接口关联的接口达到限流条件时，开启对指定接口开启限流。

第1步:配置限流规则, 将流控模式设置为关联，关联资源设置为的 /sentinel/message2。

![image-20240219151212695](typora-user-images/image-20240219151212695.png)

第2步:通过postman软件向/sentinel/message2连续发送请求，注意QPS一定要大于3

![image-20240219152607457](typora-user-images/image-20240219152607457.png)



第3步:访问/sentinel/message1,会发现已经被限流

![image-20240219152638390](typora-user-images/image-20240219152638390.png)

**链路流控模式**

链路流控模式指的是，当从某个接口过来的资源达到限流条件时，开启限流。它的功能有点类似于针对 来源配置项，区别在于:**针对来源是针对上级微服务，而链路流控是针对上级接口，也就是说它的粒度 更细。**



第1步: 编写一个service，在里面添加一个方法message

~~~java
@Service
public class TestSentinelMessage3ServiceImpl {
    @SentinelResource("message")
    public void message() {
        System.out.println("message");
    }
}
~~~



第2步: 在Controller中声明两个方法，分别调用service中的方法

~~~java

@RestController
public class TestSentinelController {

    @Autowired
    private TestSentinelMessage3ServiceImpl testSentinelMessage3Service;

    @RequestMapping("/sentinel/message1")
    public String message1() {
        testSentinelMessage3Service.message();
        return "message1";
    }
    @RequestMapping("/sentinel/message2")
    public String message2() {
        testSentinelMessage3Service.message();
        return "message2";
    }

}
~~~





![image-20240219154032740](typora-user-images/image-20240219154032740.png)

 分别通过 /sentinel/message1 和 /sentinel/message2 访问, 发现2没问题, 1的被限流了

#### 配置流控效果

**快速失败(默认):** 直接失败，抛出异常，不做任何额外的处理，是最简单的效果

**Warm Up:**它从开始阈值到最大QPS阈值会有一个缓冲阶段，一开始的阈值是最大QPS阈值的 1/3，然后慢慢增长，直到最大阈值，适用于将
突然增大的流量转换为缓步增长的场景。

**排队等待:**让请求以均匀的速度通过，单机阈值为每秒通过数量，其余的排队等待; 它还会让设 置一个超时时间，当请求超过超时间时间还未处理，则会被丢弃

### 降级规则 降级规则就是设置当满足什么条件的时候，对服务进行降级。Sentinel提供了三个衡量条件:



#### 慢调用比例 (SLOW_REQUEST_RATIO)

选择以慢调用比例作为阈值，需要设置允许的慢调用 RT（即最大的响应时间），请求的响应时间大于该值则统计为慢调用。

当单位统计时长（statIntervalMs）内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。（默认1秒内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。）

> 经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。

- 最大 RT（响应时间）：200意思是 在200毫秒处理完这个请求
- 比例阀值：0~1之间。
- `慢调用比例`：例如1秒内请求10次，8次是慢请求，则慢请求比例 0.8
- java 接口响应时间为睡眠400毫秒



根据上述可以看出触发必要条件后才会降级：

1. 请求时间 > RT
2. 单位统计时长（statIntervalMs）> 最小请求数
3. 慢比例调用比例 > 比例阀值（maxSlowRequestRatio）
4. 触发熔断后进入探测恢复状态（HALF-OPEN 状态），下一个请求响应时间小于设置的慢调用 RT 则结束熔断否则会再次被熔断

如下是我的一次降级调用，接口中休眠400毫秒，1s内连续请求5次后

![image-20240220094227461](typora-user-images/image-20240220094227461.png)



如下是我多次往复测试，不太好测试。需要参考 `经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。`

请求被熔断



![image-20240220095107292](typora-user-images/image-20240220095107292.png)

#### 异常比例 (ERROR_RATIO)

当单位统计时长（statIntervalMs 以 s 为单位）内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。

经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。

异常比率的阈值范围是 [0.0, 1.0]，代表 0% - 100%。

同慢调用比例相似，逻辑相似

![image-20240220095645324](typora-user-images/image-20240220095645324.png)

![image-20240220101010503](typora-user-images/image-20240220101010503.png)

#### 异常数 (ERROR_COUNT)

当单位统计时长（1s）内的异常数目超过阈值之后会自动进行熔断。

经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。

注意由于统计时间窗口是分钟级别的，若timeWindow小于 60s，则结束熔断状态后仍可能再进入熔断状态；

同慢调用比例相似，逻辑相似

![image-20240220101429844](typora-user-images/image-20240220101429844.png)

代码中未传递参数时可以看出异常概率应该在50%左右，从图中前部分可以看出，平均请求未出现熔断，因为请求间隔时间较长，当后半部分，1秒内多次请求时出现熔断了，老版本是以分钟来统计的，新版是秒为单位



![image-20240220101701432](typora-user-images/image-20240220101701432.png)



### 热点规则

热点参数流控规则是一种更细粒度的流控规则, 它允许将规则具体到参数上。

**热点规则简单使用**

1、编写代码

~~~java
@RequestMapping("/sentinel/message3")
@SentinelResource("message3")//注意这里必须使用这个注解标识,热点规则不生效 
public String message3(String name, Integer age) {
  return name + age;
}
~~~

第2步: 配置热点规则



![image-20240220102332812](typora-user-images/image-20240220102332812.png)

第3步: 分别用两个参数访问,会发现只对第一个参数限流了



![image-20240220102431979](typora-user-images/image-20240220102431979.png)

![image-20240220102457440](typora-user-images/image-20240220102457440.png)







**热点规则增强使用**

参数例外项允许对一个参数的具体值进行流控

编辑刚才定义的规则,增加参数例外项

![image-20240220102621265](typora-user-images/image-20240220102621265.png)

### 授权规则

很多时候，我们需要根据调用来源来判断该次请求是否允许放行，这时候可以使用 Sentinel 的来源 访问控制的功能。来源访问控制根据资源的请求来源(origin)限制资源是否通过:

若配置白名单，则只有请求来源位于白名单内时才可通过;
若配置黑名单，则请求来源位于黑名单时不通过，其余的请求通过。

![image-20240220102752041](typora-user-images/image-20240220102752041.png)

上面的资源名和授权类型不难理解，但是流控应用怎么填写呢?

>其实这个位置要填写的是来源标识，Sentinel提供了 RequestOriginParser 接口来处理来源。 只要Sentinel保护的接口资源被访问，Sentinel就会调用 RequestOriginParser 的实现类去解析
>访问来源。



第1步: 自定义来源处理规则



~~~java
@Component
public class RequestOriginParserDefinition implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest request) {
        String serviceName = request.getParameter("serviceName");
        return serviceName;
    } 
} 

~~~



第2步: 授权规则配置
这个配置的意思是只有serviceName=pc不能访问(黑名单)



![image-20240220103006353](typora-user-images/image-20240220103006353.png)



第3步：访问 http://localhost:18001/sentinel/message3?serviceName=pc  观察结果



![image-20240220103217414](typora-user-images/image-20240220103217414.png)

### 系统规则

系统保护规则是从应用级别的入口流量进行控制，从单台机器的总体 Load、RT、入口 QPS 、CPU 使用率和线程数五个维度监控应用数据，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

系统保护规则是应用整体维度的，而不是资源维度的，并且仅对入口流量 (进入应用的流量) 生效。

* Load(仅对 Linux/Unix-like 机器生效):当系统 load1 超过阈值，且系统当前的并发线程数超过 系统容量时才会触发系统保护。系统容量由系统的 maxQps * minRt 计算得出。设定参考值一般 是 CPU cores * 2.5。
* RT:当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
* 线程数:当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
* 入口 QPS:当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。
* CPU使用率:当单台机器上所有入口流量的 CPU使用率达到阈值即触发系统保护。



**扩展: 自定义异常返回**

~~~java


//异常处理页面
@Component
public class ExceptionHandlerPage implements BlockExceptionHandler {
    //BlockException 异常接口,包含Sentinel的五个异常
    // FlowException 限流异常
    // DegradeException 降级异常
    // ParamFlowException 参数限流异常
    // AuthorityException 授权异常
    // SystemBlockException 系统负载异常


    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException ex) throws Exception {
        String msg = null;
        if (ex instanceof FlowException) {
            msg = "限流了";
        } else if (ex instanceof DegradeException) {
            msg = "降级了";
        } else if (ex instanceof ParamFlowException) {
            msg = "热点参数限流";
        } else if (ex instanceof SystemBlockException) {
            msg = "系统规则（负载/...不满足要求）";
        } else if (ex instanceof AuthorityException) {
            msg = "授权规则不通过";
        }
        // http状态码
        response.setStatus(500);
        response.setCharacterEncoding("utf-8");
        response.setHeader("Content-Type", "application/json;charset=utf-8");
        response.setContentType("application/json;charset=utf-8");
        // spring mvc自带的json操作工具，叫jackson
        //返回json数据
        response.setStatus(200);
        response.setCharacterEncoding("utf-8");
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        //springmvc 的一个json转换类 （jackson）
        new ObjectMapper().writeValue(response.getWriter(), msg);
        //重定向
        //response.sendRedirect("http://www.baidu.com");
    }
}
~~~

注意:最近在学习SpringCloudAlibaba时候，需要sentinel进行流量管理控制，在配置统一处理返回异常时候，实现 UrlBlockHandler 这个接口直接爆红，原因是我使用的sentinel是2.2.5.RELEASE，官方改成了BlockExceptionHandler这个接口与实现





## @SentinelResource的使用

在定义了资源点之后，我们可以通过Dashboard来设置限流和降级策略来对资源点进行保护。同时还能
通过@SentinelResource来指定出现异常时的处理策略。

@SentinelResource 用于定义资源，并提供可选的异常处理和 fallback 配置项。其主要参数如下:



| 属性               | 作用                                                         |
| ------------------ | ------------------------------------------------------------ |
| value              | 资源名称                                                     |
| entryType          | entry类型，标记流量的方向，取值IN/OUT，默认是OUT             |
| blockHandler       | 处理BlockException的函数名称,函数要求:1. 必须是 public2.返回类型 参数与原方法一致3. 默认需和原方在同一个类中。若希望使用其他类的函数，可配置 blockHandlerClass ，并指定blockHandlerClass里面的方法。 |
| blockHandlerClass  | 存放blockHandler的类,对应的处理函数必须static修饰            |
| fallback           | 用于在抛出异常的时候提供fallback处理逻辑。fallback函数可以针对所 有类型的异常(除了 exceptionsToIgnore 里面排除掉的异常类型)进 行处理。函数要求:1. 返回类型与原方法一致 2. 参数类型需要和原方法相匹配3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置 fallbackClass ，并指定fallbackClass里面的方法。 |
| fallbackClass      | 存放fallback的类。对应的处理函数必须static修饰。             |
| defaultFallback    | 用于通用的 fallback 逻辑。默认fallback函数可以针对所有类型的异常进 行处理。若同时配置了 fallback 和 defaultFallback，以fallback为准。函 数要求:1. 返回类型与原方法一致 2. 方法参数列表为空，或者有一个 Throwable 类型的参数。3. 默认需要和原方法在同一个类中。若希望使用其他类的函数，可配置 fallbackClass ，并指定 fallbackClass 里面的方法。 |
| exceptionsToIgnore | 指定排除掉哪些异常。排除的异常不会计入异常统计，也不会进入 fallback逻辑，而是原样抛出 |
| exceptionsToTrace  | 需要trace的异常                                              |

**定义限流和降级后的处理方法**



~~~java

@Service
public class TestSentinelMessage3ServiceImpl {
        int i = 0;

        @SentinelResource(
                value = "message",
                blockHandler = "blockHandler",//指定发生BlockException时进入的方法
                fallback = "fallback"//指定发生Throwable时进入的方法
        )
        public String message() {
            i++;
            if (i % 3 == 0) {
                throw new RuntimeException();
            }
            return "message";
        }

        //BlockException时进入的方法
        public String blockHandler(BlockException ex) {
            return "接口被限流或者降级了...";
        }

        //Throwable时进入的方法
        public String fallback(Throwable throwable) {
            return "接口发生异常了...";
        }
    

~~~

**将限流和降级方法外置到单独的类中**

~~~java
@Service
@Slf4j
public class TestSentinelMessage3ServiceImpl {
    int i = 0;

    @SentinelResource(
            value = "message",
            blockHandlerClass = TestSentinelMessage3ServiceImplBlockHandlerClass.class,
            blockHandler = "blockHandler",
            fallbackClass = TestSentinelMessage3ServiceImplFallbackClass.class,
            fallback = "fallback"
    )
    public String message() {
        i++;
        if (i % 3 == 0) {
            throw new RuntimeException();
        }
        return "message4";
    }
}

@Slf4j
public class TestSentinelMessage3ServiceImplBlockHandlerClass { //注意这里必须使用static修饰方法
    public static String blockHandler(BlockException ex) {
        log.error("{}", ex);
        return "接口被限流或者降级了...";
    }
}

@Slf4j
public class TestSentinelMessage3ServiceImplFallbackClass { //注意这里必须使用static修饰方法
    public static String fallback(Throwable throwable) {
        log.error("{}", throwable);
        return "接口发生异常了...";
    }
} 


~~~



## Sentinel规则持久化



通过前面的讲解，我们已经知道，可以通过Dashboard来为每个Sentinel客户端设置各种各样的规 则，但是这里有一个问题，就是这些规则默认是存放在内存中，极不稳定，所以需要将其持久化。

### Sentinel规则的推送有下面三种模式:

| 推送模式 | 说明                                                         | 有点                         | 缺点                                                         |
| -------- | ------------------------------------------------------------ | ---------------------------- | ------------------------------------------------------------ |
| 原始模式 | API将规则推送至客户端并直接更新到内存中                      | 简单，无任何依赖             | 不保证一致性；规则保存在内存中，重启即消失。严重不建议用于生产环境 |
| Pull模式 | 扩展写数据源（WritableDataSource），客户端主动向某个规则管理中心定期轮询拉取规则，这个规则中心可以是RDBMS、文件等 | 简单，无任何依赖；规则持久化 | 不保证一致性；实时性不保证，拉取过于频繁也可能会有性能问题。 |
| Push模式 | 扩展读数据源（ReadableDataSource），规则中心统一推送，客户端通过注册监听器的方式时刻监听变化，比如使用Nacos、Zookeeper等配置中心。这种方式有更好的实时性和一致性保证。生产环境下一般采用push模式的数据源。 | 规则持久化；一致性；快速     | 引入第三方依赖                                               |



#### 原始模式

如果不做任何修改，Dashboard的推送规则方式是通过API将规则推送至客户端并直接更新到内存中：

![img](typora-user-images/a7002eb5f5b8429c8b8728b9f2e3e10d.png)

这种做法的好处是简单，无依赖；坏处是应用重启规则就会消失，仅用于简单测试，不能用于生产环境。

#### 拉模式

pull模式的数据源（如本地文件、RDBMS 等）一般是可写入的。使用时需要在客户端注册数据源：将对应的读数据源注册至对应的 RuleManager，将写数据源注册至transport的WritableDataSourceRegistry中。



![img](typora-user-images/afb25fa106bada4b37da64e902060269.png)



首先Sentinel控制台通过API将规则推送至客户端并更新到内存中，接着注册的写数据源会将新的规则保存到本地的文件中。使用 pull模式的数据源时一般不需要对Sentinel控制台进行改造。这种实现方法好处是简单，坏处是无法保证监控数据的一致性。

具体使用方式如下：

引入依赖：

~~~xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-extension</artifactId>
</dependency>
~~~

实现InitFunc接口，在init中处理DataSource初始化逻辑，并利用spi机制实现加载。



~~~java

public class FileDataSourceInit implements InitFunc {

    private static final String RULE_FILE_PATH = System.getProperty("user.home") + File.separator;

    private static final String FLOW_RULE_FILE_NAME = "FlowRule.json";

    @Override
    public void init() throws Exception {

        //处理流控规则逻辑
        dealFlowRules();
    }


    private void dealFlowRules() throws FileNotFoundException {
        String ruleFilePath = RULE_FILE_PATH + FLOW_RULE_FILE_NAME;

        //创建流控规则的可读数据源
        FileRefreshableDataSource flowRuleRDS = new FileRefreshableDataSource(
                ruleFilePath, source -> JSON.parseObject((String) source,
                new TypeReference<List<FlowRule>>() {
                })
        );

        // 将可读数据源注册至FlowRuleManager 这样当规则文件发生变化时，就会更新规则到内存
        FlowRuleManager.register2Property(flowRuleRDS.getProperty());

        WritableDataSource<List<FlowRule>> flowRuleWDS = new FileWritableDataSource<>(
                ruleFilePath, this::encodeJson
        );

        // 将可写数据源注册至 transport 模块的 WritableDataSourceRegistry 中.
        // 这样收到控制台推送的规则时，Sentinel 会先更新到内存，然后将规则写入到文件中.
        WritableDataSourceRegistry.registerFlowDataSource(flowRuleWDS);
    }


    private <T> String encodeJson(T t) {
        return JSON.toJSONString(t);
    }

}
~~~



在META-INF/services目录下创建com.alibaba.csp.sentinel.init.InitFunc，内容如下：

~~~
com.lison.springcloudservice.config.sentinel.FileDataSourceInit
~~~

这样当在Dashboard中修改了配置后，Dashboard会调用客户端的接口修改客户端内存中的值，同时将配置写入文件FlowRule.json中，这样操作的话规则是实时生效的，如果是直接修改FlowRule.json的内容，这样需要等定时任务3秒后执行才能读到最新的规则。



#### 推模式

生产环境下一般更常用的是push模式的数据源。对于push模式的数据源，如远程配置中心（ZooKeeper, Nacos, Apollo等等），推送的操作不应由Sentinel客户端进行，而应该经控制台统一进行管理，直接进行推送，数据源仅负责获取配置中心推送的配置并更新到本地。因此推送规则正确做法应该是配置中心控制台/Sentinel控制台 → 配置中心 → Sentinel数据源 → Sentinel，而不是经Sentinel数据源推送至配置中心。这样的流程就非常清晰了：

![img](typora-user-images/c5abda0a657e59634fb07442a84bf5e0.png)



### 基于Nacos配置中心控制台实现推送

配置中心控制台 → 配置中心 → Sentinel数据源 → Sentinel



引入依赖：

~~~
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>

~~~

配置文件增加nacos的数据源：

~~~yaml
spring:
  application:
    name: spring-cloud-service
  cloud:
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080
      datasource:
        flow-ds:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: ${spring.application.name}-flow
            groupId: DEFAULT_GROUP
            data-type: json
            rule-type: flow

~~~

这样直接在Nacos控制台修改规则就能实时生效了，缺点是直接在Sentinel Dashboard中修改规则配置，配置中心的配置不会发生变化。

思考：如何实现将通过sentinel控制台设置的规则直接持久化到nacos配置中心？

方法一：微服务增加基于Nacos的写数据源（WritableDataSource），发布配置到nacos配置中心。

~~~java
//核心逻辑： 实现WritableDataSource#write方法，发布配置到nacos配置中心
@Override
public void write(T t) throws Exception {
    lock.lock();
    try {
        configService.publishConfig(dataId, groupId, this.configEncoder.convert(t), ConfigType.JSON.getType());
    } finally {
        lock.unlock();
    }
}

~~~

方法二：Sentinel Dashboard监听Nacos配置的变化，如发生变化就更新本地缓存。在Sentinel Dashboard端新增或修改规则配置在保存到内存的同时，直接发布配置到nacos配置中心；Sentinel Dashboard直接从nacos拉取所有的规则配置。Sentinel Dashboard和微服务不直接通信，而是通过nacos配置中心获取到配置的变更，也就是下面的基于Sentinel控制台实现推送。



AbstractDataSourceProperties
SentinelProperties内部提供了TreeMap类型的datasource属性用于配置数据源信息。

> com.alibaba.cloud.sentinel.datasource.config.AbstractDataSourceProperties#postRegister



~~~java
public void postRegister(AbstractDataSource dataSource) {
    switch (this.getRuleType()) {
    case FLOW:
        FlowRuleManager.register2Property(dataSource.getProperty());
        break;
    case DEGRADE:
        DegradeRuleManager.register2Property(dataSource.getProperty());
        break;
    case PARAM_FLOW:
        ParamFlowRuleManager.register2Property(dataSource.getProperty());
        break;
    case SYSTEM:
        SystemRuleManager.register2Property(dataSource.getProperty());
        break;
    case AUTHORITY:
        AuthorityRuleManager.register2Property(dataSource.getProperty());
        break;
    case GW_FLOW:
        GatewayRuleManager.register2Property(dataSource.getProperty());
        break;
    case GW_API_GROUP:
        GatewayApiDefinitionManager.register2Property(dataSource.getProperty());
        break;
    default:
        break;
    }
}
~~~



**NacosDataSource从Nacos读取配置**

NacosDataSource主要负责与Nacos进行通信，实时获取Nacos的配置。



~~~java
public NacosDataSource(final Properties properties, final String groupId, final String dataId,
                       Converter<String, T> parser) {
    super(parser);
    if (StringUtil.isBlank(groupId) || StringUtil.isBlank(dataId)) {
        throw new IllegalArgumentException(String.format("Bad argument: groupId=[%s], dataId=[%s]",
            groupId, dataId));
    }
    AssertUtil.notNull(properties, "Nacos properties must not be null, you could put some keys from PropertyKeyConst");
    this.groupId = groupId;
    this.dataId = dataId;
    this.properties = properties;
    this.configListener = new Listener() {
        @Override
        public Executor getExecutor() {
            return pool;
        }

        @Override
        public void receiveConfigInfo(final String configInfo) {
            // 配置发送变更
            RecordLog.info("[NacosDataSource] New property value received for (properties: {}) (dataId: {}, groupId: {}): {}",
                properties, dataId, groupId, configInfo);
            T newValue = NacosDataSource.this.parser.convert(configInfo);
            // Update the new value to the property.
            getProperty().updateValue(newValue);
        }
    };
    // 监听配置
    initNacosListener();
    // 第一次读取配置
    loadInitialConfig();
}

private void loadInitialConfig() {
    try {
        T newValue = loadConfig();
        if (newValue == null) {
            RecordLog.warn("[NacosDataSource] WARN: initial config is null, you may have to check your data source");
        }
        getProperty().updateValue(newValue);
    } catch (Exception ex) {
        RecordLog.warn("[NacosDataSource] Error when loading initial config", ex);
    }
}

private void initNacosListener() {
    try {
        this.configService = NacosFactory.createConfigService(this.properties);
        // Add config listener.
        configService.addListener(dataId, groupId, configListener);
    } catch (Exception e) {
        RecordLog.warn("[NacosDataSource] Error occurred when initializing Nacos data source", e);
        e.printStackTrace();
    }
}

~~~

**SentinelDataSourceHandler注入NacosDataSource**
SentinelAutoConfiguration中注入了SentinelDataSourceHandler。

SentinelDataSourceHandler负责遍历配置文件中配置的DataSource，然后注入到spring容器中。

> com.alibaba.cloud.sentinel.custom.SentinelDataSourceHandler#afterSingletonsInstantiated



~~~java
public void afterSingletonsInstantiated() {
    this.sentinelProperties.getDatasource().forEach((dataSourceName, dataSourceProperties) -> {
        try {
            List<String> validFields = dataSourceProperties.getValidField();
            if (validFields.size() != 1) {
                log.error("[Sentinel Starter] DataSource " + dataSourceName + " multi datasource active and won't loaded: " + dataSourceProperties.getValidField());
                return;
            }

            AbstractDataSourceProperties abstractDataSourceProperties = dataSourceProperties.getValidDataSourceProperties();
            abstractDataSourceProperties.setEnv(this.env);
            abstractDataSourceProperties.preCheck(dataSourceName);
            this.registerBean(abstractDataSourceProperties, dataSourceName + "-sentinel-" + (String)validFields.get(0) + "-datasource");
        } catch (Exception var5) {
            log.error("[Sentinel Starter] DataSource " + dataSourceName + " build error: " + var5.getMessage(), var5);
        }

    });
}

~~~



基于Sentinel控制台实现推送
配置中心控制台 → 配置中心 → Sentinel数据源 → Sentinel

从Sentinel1.4.0开始，Sentinel控制台提供DynamicRulePublisher和DynamicRuleProvider接口用于实现应用维度的规则推送和拉取：

* DynamicRuleProvider: 拉取规则
* DynamicRulePublisher: 推送规则

可以参考Sentinel Dashboard test包下的流控规则拉取和推送的实现逻辑：



![image-20240220144907721](typora-user-images/image-20240220144907721.png)

这里主要改造Dashboard端，客户端还是采用前面的配置。

引入nacos的依赖



~~~java
 <dependency>
   <groupId>com.alibaba.nacos</groupId>
   <artifactId>nacos-client</artifactId>
   <version>2.0.3</version>
</dependency>
~~~



NacosConfig负责注入一些最基本的配置：

~~~java

package com.alibaba.csp.sentinel.dashboard.rule.nacos;

import java.util.List;

import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
import com.alibaba.csp.sentinel.datasource.Converter;
import com.alibaba.fastjson.JSON;
import com.alibaba.nacos.api.config.ConfigFactory;
import com.alibaba.nacos.api.config.ConfigService;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author Eric Zhao
 * @since 1.4.0
 */
@Configuration
public class NacosConfig {

    @Bean
    public Converter<List<FlowRuleEntity>, String> flowRuleEntityEncoder() {
        return JSON::toJSONString;
    }

    @Bean
    public Converter<String, List<FlowRuleEntity>> flowRuleEntityDecoder() {
        return s -> JSON.parseArray(s, FlowRuleEntity.class);
    }

    @Bean
    public ConfigService nacosConfigService() throws Exception {
        return ConfigFactory.createConfigService("localhost");
    }
}

~~~

FlowRuleNacosProvider负责从Nacos读取配置：

~~~java
package com.alibaba.csp.sentinel.dashboard.rule.nacos;

import java.util.ArrayList;
import java.util.List;

import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
import com.alibaba.csp.sentinel.dashboard.rule.DynamicRuleProvider;
import com.alibaba.csp.sentinel.datasource.Converter;
import com.alibaba.csp.sentinel.util.StringUtil;
import com.alibaba.nacos.api.config.ConfigService;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * @author Eric Zhao
 * @since 1.4.0
 */
@Component("flowRuleNacosProvider")
public class FlowRuleNacosProvider implements DynamicRuleProvider<List<FlowRuleEntity>> {

    @Autowired
    private ConfigService configService;
    @Autowired
    private Converter<String, List<FlowRuleEntity>> converter;

    @Override
    public List<FlowRuleEntity> getRules(String appName) throws Exception {
        String rules = configService.getConfig(appName + NacosConfigUtil.FLOW_DATA_ID_POSTFIX,
            NacosConfigUtil.GROUP_ID, 3000);
        if (StringUtil.isEmpty(rules)) {
            return new ArrayList<>();
        }
        return converter.convert(rules);
    }
}

~~~



FlowRuleNacosPublisher负责将配置写入Nacos：



~~~java

package com.alibaba.csp.sentinel.dashboard.rule.nacos;

import java.util.List;

import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
import com.alibaba.csp.sentinel.dashboard.rule.DynamicRulePublisher;
import com.alibaba.csp.sentinel.datasource.Converter;
import com.alibaba.csp.sentinel.util.AssertUtil;
import com.alibaba.nacos.api.config.ConfigService;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * @author Eric Zhao
 * @since 1.4.0
 */
@Component("flowRuleNacosPublisher")
public class FlowRuleNacosPublisher implements DynamicRulePublisher<List<FlowRuleEntity>> {

    @Autowired
    private ConfigService configService;
    @Autowired
    private Converter<List<FlowRuleEntity>, String> converter;

    @Override
    public void publish(String app, List<FlowRuleEntity> rules) throws Exception {
        AssertUtil.notEmpty(app, "app name cannot be empty");
        if (rules == null) {
            return;
        }
        configService.publishConfig(app + NacosConfigUtil.FLOW_DATA_ID_POSTFIX,
            NacosConfigUtil.GROUP_ID, converter.convert(rules));
    }
}

~~~

上面都是新增的类，最后还需要在Dashboard查询和修改规则时进行修改，具体修改是在FlowControllerV2



![image-20240221092028120](typora-user-images/image-20240221092028120.png)



以 Nacos 为例，若希望使用 Nacos 作为动态规则配置中心，用户可以提取出相关的类，然后只需在 FlowControllerV2 中指定对应的 bean 即可开启 Nacos 适配。前端页面需要手动切换，或者修改前端路由配置（sidebar.html 流控规则路由从 dashboard.flowV1 改成 dashboard.flow 即可，注意簇点链路页面对话框需要自行改造）。

~~~java
@Autowired
@Qualifier("flowRuleNacosProvider")
private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;
@Autowired
@Qualifier("flowRuleNacosPublisher")
private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;

~~~



**修改控制台源码实现流控规则持久化**

接下来，参考以上官方提供的解决方案，我们来实际操作一下

**1、改造代码**

首先将pom中的sentinel-datasource-nacos中的scope去掉，将Nacos相关依赖引入到编译环境中来。

~~~xml
  <!-- for Nacos rule publisher sample -->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
~~~



![image-20240221091728143](typora-user-images/image-20240221091728143.png)



将test目录下nacos动态规则实现的相关代码，复制到com.alibaba.csp.sentinel.dashboard.rule包下

![image-20240221091837069](typora-user-images/image-20240221091837069.png)



修改FlowControllerV2类，将动态规则发布及拉取的注入类，替换为flowRuleNacosProvider及flowRuleNacosPublisher。



![image-20240221091441127](typora-user-images/image-20240221091441127.png)



**2、改造页面**

找到图中目录下的sidebar页面，将流控规则菜单中的dashboard.flowV1改为dashboard.flow。

![image-20240221092353574](typora-user-images/image-20240221092353574.png)



![image-20240221092419244](typora-user-images/image-20240221092419244.png)

![image-20240221092834785](typora-user-images/image-20240221092834785.png)

以流控规则测试，当在sentinel dashboard配置了流控规则，会在nacos配置中心生成对应的配置，这样客户端就能读取到这个流控规则配置了。

![image-20240221093021265](typora-user-images/image-20240221093021265.png)



spring-cloud-service.json

~~~json
[
    {
        "app":"spring-cloud-service",
        "clusterConfig":{
            "acquireRefuseStrategy":0,
            "clientOfflineTime":2000,
            "fallbackToLocalWhenFail":true,
            "resourceTimeout":2000,
            "resourceTimeoutStrategy":0,
            "sampleCount":10,
            "strategy":0,
            "thresholdType":0,
            "windowIntervalMs":1000
        },
        "clusterMode":false,
        "controlBehavior":0,
        "count":5,
        "gmtCreate":1708478952132,
        "gmtModified":1708478952132,
        "grade":1,
        "id":9,
        "ip":"10.108.202.7",
        "limitApp":"default",
        "port":9998,
        "resource":"/sentinel/message3",
        "strategy":0
    }
]
~~~





**进阶：**

**此时你启动nacos-dashboard，正要到流控规则页面进行尝试时，你会发现**

![image-20240221093711599](typora-user-images/image-20240221093711599.png)

**有个回到单机页面的按钮，你好奇的点了一下，满怀期待的进行配置，但是却发现配置不能生效，这是因为单机页面的执行的方法还是默认的方法，需要进行如下修改：**



>
>
>resources/app/views/flow_v2.html  
>
>两种方法：1.进到这个页面，找到执行的方法修改为自定义的V2类下的方法
>
>​         2.注释掉按钮
>
>为了方便快捷，我们直接注释

![image-20240221093819873](typora-user-images/image-20240221093819873.png)



***为了方便我们以后的配置，更为牛逼的进阶之旅开启，快上车***

 一般我们习惯从簇点链路直接配置流控，而不是到流控规则页面进行配置，但是问题来了，从簇点链路进行配置的不生效，按F12看请求会发现，他还是请求的 /v1/flow 而不是 /v2/flow



![image-20240221094018816](typora-user-images/image-20240221094018816.png)

解决问题：

>resources/app/scripts/controllers/identity.js  
>
>对比这修改，至于为什么这么改不再赘述，有兴趣的可以对比一下两个路径执行的方法 



![image-20240221094225094](typora-user-images/image-20240221094225094.png)



![image-20240221094244920](typora-user-images/image-20240221094244920.png)

修改到这，你会发现从簇点链路配置的流程规则可以推送到nacos了，但是新问题，出现了，保存完后会自动跳转到展示页面，但是展示页面是空的~~~~~~~~~~~~~~

原因：F12查看请求得知，查询方法还是执行的V1版本的默认方法，而不是我们自定义的V2里面的方法，继续在当前js文件进行修改



![image-20240221094447664](typora-user-images/image-20240221094447664.png)

![image-20240221094852671](typora-user-images/image-20240221094852671.png)

终于实现了分别在两个页面进行流控配置

![image-20240221095745495](typora-user-images/image-20240221095745495.png)

![image-20240221155333042](typora-user-images/image-20240221155333042.png)
