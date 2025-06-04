---
title: Spring Cloud Alibaba-07-RocketMQ消息驱动
date: 2025-04-07 19:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2024.4.20`

# Spring Cloud Alibaba-07-RocketMQ消息驱动

[toc]



##  MQ简介

消息队列（Message Queue）简称 MQ，是一种跨进程的通信机制，通常用于应用程序间进行数据的异步传输，MQ 产品在架构中通常也被叫作“消息中间件”。它的最主要职责就是保证服务间进行可靠的数据传输，同时实现服务间的解耦。



![image-20240507111933014](typora-user-images/image-20240507111933014.png)





### MQ的应用场景

* 应用解耦

  >在电商平台中，用户下订单需要调用订单系统，此时订单系统还需要调用库存系统、支付系统、物流系统完成业务。此时会产生两个问题：
  >
  >1 如果库存系统出现故障，会造成整个订单系统崩溃。
  >
  >2 如果需求修改，新增了一个X系统，此时必须修改订单系统的代码。
  >
  >![image-20230627191528034](typora-user-images/image-20230627191528034.png)
  >
  >
  >
  >如果在系统中引入MQ，即订单系统将消息先发送到MQ中，MQ再转发到其他系统，则会解决以下问题： 
  >
  >1、由于订单系统只发消息给MQ，不直接对接其他系统，如果库存系统出现故障，不影响整个订单。
  >
  >2、如果需求修改，新增了一个X系统，此时无需修改订单系统的代码，只需修改MQ将消息发送给X系统即可。
  >
  >![image-20240507112405845](typora-user-images/image-20240507112405845.png)

  

  

* 异步提速

  >如果订单系统同步访问每个系统，则用户下单等待时长如下：
  >
  >![image-20230628093823674](typora-user-images/image-20230628093823674.png)
  >
  >
  >
  >如果引入MQ,则用户等待时间如下：
  >
  >![image-20230628094058045](typora-user-images/image-20230628094058045.png)

  

* 消峰填谷

  > 假设我们的系统每秒只能承载1000请求，如果请求瞬间增多到每秒 5000，则会造成系统崩溃。此时引入mq即可解决该问题
  >
  > ![image-20230628094428585](typora-user-images/image-20230628094428585.png)
  >
  > 使用了MQ之后，限制消费消息的速度为1000，这样一来，高峰期产生的数据势必会被积压在MQ中，高峰就被“削”掉了，但是因为消息积压，在高峰期过后的一段时间内，消费消息的速度还是会维持 在1000，直到消费完积压的消息，这就叫做“填谷”。 
  >
  > ![image-20230628094521252](typora-user-images/image-20230628094521252.png)
  >
  > 流量削峰也是消息队列 MQ 的常用场景，一般在秒杀或团队抢购(高并发)活动中使用广泛。
  > 在秒杀或团队抢购活动中，由于用户请求量较大，导致流量暴增，秒杀的应用在处理如此大量的访问流 量后，下游的通知系统无法承载海量的调用量，甚至会导致系统崩溃等问题而发生漏通知的情况。为解 决这些问题，可在应用和下游通知系统之间加入消息队列 MQ。
  >
  > 秒杀处理流程如下所述:
  >
  > 1. 用户发起海量秒杀请求到秒杀业务处理系统。
  > 2. 秒杀处理系统按照秒杀处理逻辑将满足秒杀条件的请求发送至消息队列 MQ。
  > 3. 下游的通知系统订阅消息队列 MQ 的秒杀相关消息，再将秒杀成功的消息发送到相应用户。
  > 4. 用户收到秒杀成功的通知。

  



### 常见的MQ产品

目前业界有很多MQ产品，比较出名的有下面这些:

**ZeroMQ**

号称最快的消息队列系统，尤其针对大吞吐量的需求场景。扩展性好，开发比较灵活，采用C语言 实现，实际上只是一个socket库的重新封装，如果做为消息队列使用，需要开发大量的代码。 ZeroMQ仅提供非持久性的队列，也就是说如果down机，数据将会丢失。

**RabbitMQ**

使用erlang语言开发，性能较好，适合于企业级的开发。但是不利于做二次开发和维护。 ActiveMQ
历史悠久的Apache开源项目。已经在很多产品中得到应用，实现了JMS1.1规范，可以和spring- jms轻松融合，实现了多种协议，支持持久化到数据库，对队列数较多的情况支持不好。

**RocketMQ**

阿里巴巴的MQ中间件，由java语言开发，性能非常好，能够撑住双十一的大流量，而且使用起来 很简单。

**Kafka**

Kafka是Apache下的一个子项目，是一个高性能跨语言分布式Publish/Subscribe消息队列系统， 相对于ActiveMQ是一个非常轻量级的消息系统，除了性能非常好之外，还是一个工作良好的分布 式系统。



### RocketeMQ的架构及概念



RocketMQ 有很多优秀的特性，在可用性方面，RocketMQ 强调集群无单点，任意一点高可用，客户端具备负载均衡能力，可以轻松实现水平扩容；在性能方面，在天猫双 11 大促背后的亿级消息处理就是通过 RocketMQ 提供的保障；在 API 方面，提供了丰富的功能，可以实现异步消息、同步消息、顺序消息、事务消息等丰富的功能，能满足大多数应用场景；在可靠性方面，提供了消息持久化、失败重试机制、消息查询追溯的功能，进一步为可靠性提供保障。



![在这里插入图片描述](typora-user-images/72746e97a2cd474bb21601f507779a25.png)



了解 RocketMQ 的诸多特性后，咱们来理解 RocketMQ 几个重要的概念：

* 消息模型（Message Model）：RocketMQ主要由 Producer、Broker、Consumer 三部分组成，其中Producer 负责生产消息，Consumer 负责消费消息，Broker 负责存储消息。Broker 在实际部署过程中对应一台服务器，每个 Broker 可以存储多个Topic的消息，每个Topic的消息也可以分片存储于不同的 Broker。Message Queue 用于存储消息的物理地址，每个Topic中的消息地址存储于多个 Message Queue 中。ConsumerGroup 由多个Consumer 实例构成。
* 消息生产者（Producer）：负责生产消息，一般由业务系统负责生产消息。一个消息生产者会把业务应用系统里产生的消息发送到broker服务器。RocketMQ提供多种发送方式，同步发送、异步发送、顺序发送、单向发送。同步和异步方式均需要Broker返回确认信息，单向发送不需要。
* 消息消费者（Consumer）：负责消费消息，一般是后台系统负责异步消费。一个消息消费者会从Broker服务器拉取消息、并将其提供给应用程序。从用户应用的角度而言提供了两种消费形式：拉取式消费、推动式消费。
* 生产者组（Producer Group）：同一类Producer的集合，这类Producer发送同一类消息且发送逻辑一致。如果发送的是事物消息且原始生产者在发送之后崩溃，则Broker服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。
* 消费者组（Consumer Group）：同一类Consumer的集合，这类Consumer通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现负载均衡和容错的目标变得非常容易。要注意的是，消费者组的消费者实例必须订阅完全相同的Topic。RocketMQ 支持两种消息模式：集群消费（Clustering）和广播消费（Broadcasting）。
* 代理服务器（Broker Server）：消息中转角色，负责存储消息、转发消息。代理服务器在RocketMQ系统中负责接收从生产者发送来的消息并存储、同时为消费者的拉取请求作准备。代理服务器也存储消息相关的元数据，包括消费者组、消费进度偏移和主题和队列消息等。
* 名字服务（Name Server）：名称服务充当路由消息的提供者。生产者或消费者能够通过名字服务查找各主题相应的Broker IP列表。多个Namesrv实例组成集群，但相互独立，没有信息交换。
* 主题（Topic）：表示一类消息的集合，每个主题包含若干条消息，每条消息只能属于一个主题，是RocketMQ进行消息订阅的基本单位。
* 标签（Tag）:为消息设置的标志，用于同一主题下区分不同类型的消息。来自同一业务单元的消息，可以根据不同业务目的在同一主题下设置不同标签。标签能够有效地保持代码的清晰度和连贯性，并优化RocketMQ提供的查询系统。消费者可以根据Tag实现对不同子主题的不同消费逻辑，实现更好的扩展性。
* 消息（Message）:消息系统所传输信息的物理载体，生产和消费数据的最小单位，每条消息必须属于一个主题。RocketMQ中每个消息拥有唯一的Message ID，且可以携带具有业务标识的Key。系统提供了通过Message ID和Key查询消息的功能。
* 拉取式消费（Pull Consumer）:Consumer消费的一种类型，应用通常主动调用Consumer的拉消息方法从Broker服务器拉消息、主动权由应用控制。一旦获取了批量消息，应用就会启动消费过程。
* 推动式消费（Push Consumer）:Consumer消费的一种类型，该模式下Broker收到数据后会主动推送给消费端，该消费模式一般实时性较高。
* 集群消费（Clustering）:集群消费模式下,相同Consumer Group的每个Consumer实例平均分摊消息。
* 广播消费（Broadcasting）:广播消费模式下，相同Consumer Group的每个Consumer实例都接收全量的消息。
* 普通顺序消息（Normal Ordered Message）：普通顺序消费模式下，消费者通过同一个消费队列收到的消息是有顺序的，不同消息队列收到的消息则可能是无顺序的。
* 严格顺序消息（Strictly Ordered Message）：严格顺序消息模式下，消费者收到的所有消息均是有顺序的。



官方文档地址：https://rocketmq.apache.org/zh/docs/







## RocketMQ入门

RocketMQ 是一款分布式消息队列中间件，官方地址为http://rocketmq.apache.org/，目前最新版本为4.8.0。RocketMQ 最初设计是为了满足阿里巴巴自身业务对异步消息传递的需要，在 3.X 版本后正式开源并捐献给 Apache，目前已孵化为 Apache 顶级项目，同时也是国内使用最广泛、使用人数最多的 MQ 产品之一

![image-20240507112941777](typora-user-images/image-20240507112941777.png)





### RocketMQ环境搭建

接下来我们先在linux平台下安装一个RocketMQ的服务,本文使用 docker-compose 安装RocketMq

**创建docker文件夹**

~~~
mkdir rocketmq
cd rocketmq
mkdir data
cd data
mkdir -p brokerconf logs store
~~~



**在rocketmq文件夹下创建docker-compose.yml文件**



~~~

## rocketmq
version: '3.8'
services:
  rmqnamesrv1:
    image: apache/rocketmq:4.9.3
    container_name: rmqnamesrv1
    ports:
      - 9876:9876
    volumes:
      - /Users/lison/work/data/dockerData/rocketmq/rmqnamesrv1/data/logs:/opt/logs
      - /Users/lison/work/data/dockerData/rocketmq/rmqnamesrv1/data/store:/opt/store
    command: sh mqnamesrv  
    networks:
      - nt_dev

  rmqnamesrv2:
    image: apache/rocketmq:4.9.3
    container_name: rmqnamesrv2
    ports:
      - 9877:9876
    volumes:
      - /Users/lison/work/data/dockerData/rocketmq/rmqnamesrv2/data/logs:/opt/logs
      - /Users/lison/work/data/dockerData/rocketmq/rmqnamesrv2/data/store:/opt/store
    command: sh mqnamesrv  
    networks:
      - nt_dev         
 

  rmqbroker1:
    image: apache/rocketmq:4.9.3
    container_name: rmqbroker1
    ports:
      - 10911:10911
    volumes:
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker1/data/logs:/opt/logs
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker1/data/store:/opt/store
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker1/brokerconf/broker.conf:/etc/rocketmq/broker.conf
    environment:
        NAMESRV_ADDR: "rmqnamesrv1:9876;rmqnamesrv2:9876"
        JAVA_OPTS: " -Duser.home=/opt"
        JAVA_OPT_EXT: "-server -Xms128m -Xmx128m -Xmn128m"
    command: sh mqbroker -c /etc/rocketmq/broker.conf autoCreateTopicEnable=true &
    depends_on:
      - rmqnamesrv1
      - rmqnamesrv2
    networks:
      - nt_dev 


  rmqbroker2:
    image: apache/rocketmq:4.9.3
    container_name: rmqbroker2
    ports:
      - 10912:10911
    volumes:
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker2/data/logs:/opt/logs
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker2/data/store:/opt/store
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker2/brokerconf/broker.conf:/etc/rocketmq/broker.conf
    environment:
        NAMESRV_ADDR: "rmqnamesrv1:9876;rmqnamesrv2:9876"
        JAVA_OPTS: " -Duser.home=/opt"
        JAVA_OPT_EXT: "-server -Xms128m -Xmx128m -Xmn128m"
    command: sh mqbroker -c /etc/rocketmq/broker.conf autoCreateTopicEnable=true &
    depends_on:
      - rmqnamesrv1
      - rmqnamesrv2
    networks:
      - nt_dev 

  rmqbroker3:
    image: apache/rocketmq:4.9.3
    container_name: rmqbroker3
    ports:
      - 10913:10911
    volumes:
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker3/data/logs:/opt/logs
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker3/data/store:/opt/store
      - /Users/lison/work/data/dockerData/rocketmq/rmqbroker3/brokerconf/broker.conf:/etc/rocketmq/broker.conf
    environment:
        NAMESRV_ADDR: "rmqnamesrv1:9876;rmqnamesrv2:9876"
        JAVA_OPTS: " -Duser.home=/opt"
        JAVA_OPT_EXT: "-server -Xms128m -Xmx128m -Xmn128m"
    command: sh mqbroker -c /etc/rocketmq/broker.conf autoCreateTopicEnable=true &
    depends_on:
      - rmqnamesrv1
      - rmqnamesrv2
    networks:
      - nt_dev

 
  rmqconsole:
    image: styletang/rocketmq-console-ng
    container_name: rmqconsole
    ports:
      - 8080:8080
    environment:
        JAVA_OPTS: "-Drocketmq.namesrv.addr=rmqnamesrv1:9876;rmqnamesrv2:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false"
    depends_on:
      - rmqbroker1
      - rmqbroker2
      - rmqbroker3
    networks:
      - nt_dev
 
networks:
  nt_dev:
    external: true
    driver: bridge

  
~~~



**brokerconf下新建broker.conf文件并存储**

创建文件



/Users/lison/work/data/dockerData/rocketmq/rmqbroker1/brokerconf/broker.conf

~~~
brokerClusterName = DefaultCluster
#broker名称
brokerName = rmqbroker1
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

~~~

/Users/lison/work/data/dockerData/rocketmq/rmqbroker2/brokerconf/broker.conf

~~~
brokerClusterName = DefaultCluster
#broker名称
brokerName = rmqbroker2
brokerId = 1
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

~~~



/Users/lison/work/data/dockerData/rocketmq/rmqbroker3/brokerconf/broker.conf

~~~
brokerClusterName = DefaultCluster
#broker名称
brokerName = rmqbroker3
brokerId = 2
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

~~~





**保存上边配置，执行docker-compose**

~~~
docker-compose up -d
~~~

docker ps 查看是否都启动了，如果都启动了，在成功，如果有没有启动成功，则可以查看docker日志，一般都是，ip设置问题。

![image-20240509105505568](typora-user-images/image-20240509105505568.png)



**打开对应的对口之后可以通过浏览器控制台进行查看**

![image-20240509105539663](typora-user-images/image-20240509105539663.png)





## SpringBoot 集成 RocketMQ

1、使用Java代码来演示消息的发送和接收，加入依赖

~~~xml
 <!-- RocketMQ客户端，版本与Broker保持一致  -->
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.2.2</version>
        </dependency>
        
 <!-- 也可直接定义指定版本  
 			<dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>4.5.2</version>
        </dependency>-->
        
~~~



2、配置应用 application.yml。



~~~yaml
#rocketmq配置
rocketmq:
  enhance:
    # 启动隔离，用于激活配置类EnvironmentIsolationConfig
    # 启动后会自动在topic上拼接激活的配置文件，达到自动隔离的效果
    enabledIsolation: true
    # 隔离环境名称，拼接到topic后，topic_dev，默认空字符串
    environment: dev
  topic: springboot-mq
  name-server: 127.0.0.1:9876
  # 生产者配置
  producer:
    # 发送同一类消息的设置为同一个group，保证唯一
    group: rocketmq-pro-group
    # 发送消息超时时间,默认3000
    sendMessageTimeout: 30000
    # 发送消息失败重试次数，默认2
    retryTimesWhenSendFailed: 10
    # 异步消息重试此处，默认2
    retryTimesWhenSendAsyncFailed: 10
    # 消息最大长度 默认1024*4(4M)
    maxMessageSize: 4096
    # 是否在内部发送失败时重试另一个broker，默认false
    retryNextServer: false
    # 压缩消息阈值，默认4k(1024 * 4)
    compressMessageBodyThreshold: 4096
  consumer:
    group: rocketmq-consumer-group
~~~





~~~java
package com.ruipeng.service;
//发送短信的服务
@Slf4j
@Service
@RocketMQMessageListener(consumerGroup = "rocketmq-consumer-group", topic = "springboot-mq") 
public class SmsService implements RocketMQListener<Order> { 
	@Override 
	public void onMessage(Order order) {
	 log.info("收到一个信息{},接下来发送短信", JSON.toJSONString(order)); 
  }
   } 


~~~





~~~java
//测试
@RunWith(SpringRunner.class)
@SpringBootTest(classes = OrderApplication.class)
public class MessageTypeTest {
    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    //同步消息
    @Test
    public void testSyncSend() {
//参数一: topic， 如果想添加tag 可以使用"topic:tag"的写法 //参数二: 消息内容
        SendResult sendResult =
                rocketMQTemplate.syncSend("test-topic-1", "这是一条同步消息");
        System.out.println(sendResult);
    }

    //异步消息
    @Test
    public void testAsyncSend() throws InterruptedException {
        public void testSyncSendMsg () {
//参数一: topic, 如果想添加tag 可以使用"topic:tag"的写法
//参数二: 消息内容
//参数三: 回调函数, 处理返回结果 rocketMQTemplate.asyncSend("test-topic-1", "这是一条异步消息", new 
            SendCallback() {
                @Override
                public void onSuccess (SendResult sendResult){
                    System.out.println(sendResult);
                }
                @Override
                public void onException (Throwable throwable){
                    System.out.println(throwable);
                }
            });
//让线程不要终止 Thread.sleep(30000000); 

        }
    }

    //单向消息
    @Test
    public void testOneWay() {
        rocketMQTemplate.sendOneWay("test-topic-1", "这是一条单向消息");
    } 
}

~~~



三种发送方式的对比

| 发送方式 | 发送 TPS | 发送结果反馈 | 可靠性   |
| -------- | -------- | ------------ | -------- |
| 同步发送 | 快       | 有           | 不丢失   |
| 异步发送 | 快       | 有           | 不丢失   |
| 单向发送 | 最快     | 无           | 可能丢失 |



消费主义细节：

~~~java
@RocketMQMessageListener(
consumerGroup = "shop",//消费者分组
topic = "order-topic",//要消费的主题
consumeMode = ConsumeMode.CONCURRENTLY, //消费模式:无序和有序 messageModel = MessageModel.CLUSTERING, //消息模式:广播和集群,默认是集群 
)
public class SmsService implements RocketMQListener<Order> {}

~~~

RocketMQ支持两种消息模式:

- 广播消费: 每个消费者实例都会收到消息,也就是一条消息可以被每个消费者实例处理;
- 集群消费: 一条消息只能被一个消费者实例消费
