---
title: Spring Cloud Alibaba-09-Seata分布式事务
date: 2025-04-09 19:10:56
---




`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2024.5.03`

# Spring Cloud Alibaba-09-Seata分布式事务

[toc]

## 分布式事务基础

### 事务

事务指的就是一个操作单元，在这个操作单元中的所有操作最终要保持一致的行为，要么所有操作都成功，要么所有的操作都被撤销。简单地说，事务提供一种“要么什么都不做，要么做全套”机制。

### 本地事务

本地事物其实可以认为是数据库提供的事务机制。说到数据库事务就不得不说，数据库事务中的四 大特性:

* A:原子性(Atomicity)，一个事务中的所有操作，要么全部完成，要么全部不完成
* C:一致性(Consistency)，在一个事务执行之前和执行之后数据库都必须处于一致性状态
* I:隔离性(Isolation)，在并发环境中，当不同的事务同时操作相同的数据时，事务之间互不影响
* D:持久性(Durability)，指的是只要事务成功结束，它对数据库所做的更新就必须永久的保存下来

数据库事务在实现时会将一次事务涉及的所有操作全部纳入到一个不可分割的执行单元，该执行单元中的所有操作要么都成功，要么都失败，只要其中任一操作执行失败，都将导致整个事务的回滚

### 分布式事务

分布式事务指事务的参与者、支持事务的服务器、资源服务器以及事务管理器分别位于不同的分布
式系统的不同节点之上。

简单的说，就是一次大的操作由不同的小操作组成，这些小的操作分布在不同的服务器上，且属于不同
的应用，分布式事务需要保证这些小操作要么全部成功，要么全部失败。

本质上来说，分布式事务就是为了保证不同数据库的数据一致性。

### 分布式事务的场景

**1、单体系统访问多个数据库，一个服务需要调用多个数据库实例完成数据的增删改操作**

![image-20240514095424805](typora-user-images/image-20240514095424805.png)



**2、多个微服务访问同一个数据库实例完成数据的增删改操作**

![image-20240514095650303](typora-user-images/image-20240514095650303.png)



3、多个微服务访问多个数据库，完成数据的增删改操作

![image-20240514095835213](typora-user-images/image-20240514095835213.png)





## 分布式事务的解决方案

### 全局事务

全局事务基于DTP模型实现。DTP是由X/Open组织提出的一种分布式事务模型——X/Open Distributed Transaction Processing Reference Model。它规定了要实现分布式事务，需要三种角色:

* AP: Application 应用系统 (微服务)
* TM: Transaction Manager 事务管理器 (全局事务管理)
* RM: Resource Manager 资源管理器 (数据库)

整个事务分成两个阶段:

- 阶段一: 表决阶段，所有参与者都将本事务执行预提交，并将能否成功的信息反馈发给协调者。
- 阶段二: 执行阶段，协调者根据所有参与者的反馈，通知所有参与者，步调一致地执行提交或者回 滚。

![image-20240516094845785](typora-user-images/image-20240516094845785.png)

优点

- 提高了数据一致性的概率，实现成本较低

缺点

- 单点问题: 事务协调者宕机
- 同步阻塞: 延迟了提交时间，加长了资源阻塞时间
- 数据不一致: 提交第二阶段，依然存在commit结果未知的情况，有可能导致数据不一致



### 可靠消息服务

基于可靠消息服务的方案是通过消息中间件保证上、下游应用数据操作的一致性。假设有A和B两个 系统，分别可以处理任务A和任务B。此时存在一个业务流程，需要将任务A和任务B在同一个事务中处 理。就可以使用消息中间件来实现这种分布式事务



![image-20240520161819461](typora-user-images/image-20240520161819461.png)



**第一步: 消息由系统A投递到中间件**

1. 在系统A处理任务A前，首先向消息中间件发送一条消息
2. 消息中间件收到后将该条消息持久化，但并不投递。持久化成功后，向A回复一个确认应答
3. 系统A收到确认应答后，则可以开始处理任务A
4. 任务A处理完成后，向消息中间件发送Commit或者Rollback请求。该请求发送完成后，对系统A而 言，该事务的处理过程就结束了
5. 如果消息中间件收到Commit，则向B系统投递消息;如果收到Rollback，则直接丢弃消息。但是 如果消息中间件收不到Commit和Rollback指令，那么就要依靠"超时询问机制"。

>超时询问机制
>系统A除了实现正常的业务流程外，还需提供一个事务询问的接口，供消息中间件调 用。当消息中间件收到发布消息便开始计时，如果到了超时没收到确认指令，就会主动调用 系统A提供的事务询问接口询问该系统目前的状态。该接口会返回三种结果，中间件根据三 种结果做出不同反应:
>提交:将该消息投递给系统B
>回滚:直接将条消息丢弃
>处理中:继续等待



**第二步: 消息由中间件投递到系统B**

消息中间件向下游系统投递完消息后便进入阻塞等待状态，下游系统便立即进行任务的处理，任务
处理完成后便向消息中间件返回应答

* 如果消息中间件收到确认应答后便认为该事务处理完毕
* 如果消息中间件在等待确认应答超时之后就会重新投递，直到下游消费者返回消费成功响应为止。
  一般消息中间件可以设置消息重试的次数和时间间隔，如果最终还是不能成功投递，则需要手工干
  预。这里之所以使用人工干预，而不是使用让A系统回滚，主要是考虑到整个系统设计的复杂度问
  题。

基于可靠消息服务的分布式事务，前半部分使用异步，注重性能;后半部分使用同步，注重开发成本。



### 最大努力通知

最大努力通知也被称为定期校对，其实是对第二种解决方案的进一步优化。它引入了本地消息表来
记录错误消息，然后加入失败消息的定期校对功能，来进一步保证消息会被下游系统消费。



![image-20240521094502177](typora-user-images/image-20240521094502177.png)



**第一步: 消息由系统A投递到中间件**

1. 处理业务的同一事务中，向本地消息表中写入一条记录

2. 准备专门的消息发送者不断地发送本地消息表中的消息到消息中间件，如果发送失败则重试



**第二步: 消息由中间件投递到系统B**

1. 消息中间件收到消息后负责将该消息同步投递给相应的下游系统，并触发下游系统的任务执行
2. 当下游系统处理成功后，向消息中间件反馈确认应答，消息中间件便可以将该条消息删除，从而该
   事务完成
3. 对于投递失败的消息，利用重试机制进行重试，对于重试失败的，写入错误消息表
4. 消息中间件需要提供失败消息的查询接口，下游系统会定期查询失败消息，并将其消费

这种方式的优缺点:
   **优点:** 一种非常经典的实现，实现了最终一致性。
   **缺点:** 消息表会耦合到业务系统中，如果没有封装好的解决方案，会有很多杂活需要处理。



### TCC事务

TCC即为Try Confirm Cancel，它属于补偿型分布式事务。TCC实现分布式事务一共有三个步骤:

**Try:尝试待执行的业务**

这个过程并未执行业务，只是完成所有业务的一致性检查，并预留好执行所需的全部资源

**Confirm:确认执行业务**

确认执行业务操作，不做任何业务检查， 只使用Try阶段预留的业务资源。通常情况下，采用TCC 则认为 Confirm阶段是不会出错的。即:只要Try成功，Confirm一定成功。若Confirm阶段真的 出错了，需引入重试机制或人工处理。

**Cancel:取消待执行的业务**

取消Try阶段预留的业务资源。通常情况下，采用TCC则认为Cancel阶段也是一定成功的。若 Cancel阶段真的出错了，需引入重试机制或人工处理。



![image-20240522090312508](typora-user-images/image-20240522090312508.png)



![image-20240522090352573](typora-user-images/image-20240522090352573.png)



TCC两阶段提交与XA两阶段提交的区别是:
XA是资源层面的分布式事务，强一致性，在两阶段提交的整个过程中，一直会持有资源的锁。
TCC是业务层面的分布式事务，最终一致性，不会一直持有资源的锁。

TCC事务的优缺点:

**优点**:把数据库层的二阶段提交上提到了应用层来实现，规避了数据库层的2PC性能低下问题。
**缺点**:TCC的Try、Confirm和Cancel操作功能需业务提供，开发成本高。



## Seata介绍

2019 年 1 月，阿里巴巴中间件团队发起了开源项目 Fescar(Fast & EaSy Commit And Rollback)，其愿景是让分布式事务的使用像本地事务的使用一样，简单和高效，并逐步解决开发者们 遇到的分布式事务方面的所有难题。后来更名为 Seata，意为:Simple Extensible Autonomous Transaction Architecture，是一套分布式事务解决方案。

官方网站：https://seata.apache.org/



![image-20240522091143334](typora-user-images/image-20240522091143334.png)



Seata的设计目标是对业务无侵入，因此从业务无侵入的2PC方案着手，在传统2PC的基础上演进。 它把一个分布式事务理解成一个包含了若干分支事务的全局事务。全局事务的职责是协调其下管辖的分 支事务达成一致，要么一起成功提交，要么一起失败回滚。此外，通常分支事务本身就是一个关系数据 库的本地事务。

![image-20240522090958255](typora-user-images/image-20240522090958255.png)



Seata主要由三个重要组件组成:

**TC**:Transaction Coordinator 事务协调器，管理全局的分支事务的状态，用于全局性事务的提交 和回滚。

**TM**:Transaction Manager 事务管理器，用于开启、提交或者回滚全局事务。

**RM:**Resource Manager 资源管理器，用于分支事务上的资源管理，向TC注册分支事务，上报分 支事务的状态，接受TC的命令来提交或者回滚分支事务。

![在这里插入图片描述](typora-user-images/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3J1aXBlbmcyNTA=,size_16,color_FFFFFF,t_70-6340401.png)



**Seata的执行流程如下:**

1. **全局事务开始**：TM向TC请求开启一个全局事务，TC生成一个全局事务ID（XID）。
2. **分支事务注册**：TM通知涉及的每个RM开始分支事务，RM在执行本地事务前向TC注册分支事务，并在本地保存XID。
3. **执行业务操作**：RM在本地事务上下文中执行SQL操作，同时记录undo log（用于事务回滚）。
4. **一阶段提交预处理**：TM请求TC进行一阶段提交的预处理，TC通知所有RM准备提交，RM将本地事务状态置为预提交，并回复TC。
5. **二阶段提交**：若所有RM的预提交都成功，TM通知TC进行二阶段提交。TC命令所有RM正式提交分支事务，RM提交本地事务并清理undo log。
6. **二阶段回滚**：如果任意RM的预提交失败，或TM请求回滚，TC会命令所有RM进行分支事务的回滚，RM使用undo log恢复数据到事务前状态，并清除undo log。



**Seata实现2PC与传统2PC的差别:**

1. 架构层次方面，传统2PC方案的 RM 实际上是在数据库层，RM本质上就是数据库自身，通过XA协 议实现，而 Seata的RM是以jar包的形式作为中间件层部署在应用程序这一侧的。
2. 两阶段提交方面，传统2PC无论第二阶段的决议是commit还是rollback，事务性资源的锁都要保 持到Phase2完成才释放。而Seata的做法是在Phase1 就将本地事务提交，这样就可以省去Phase2 持锁的时间，整体提高效率。



## Docker安装Seata-Nacos注册中心，DB存储

### 准备工作

**1、生成seata配置文件**

我们通过创建临时容器的方式，直接从中拷贝出自动生成的配置信息，待挂载使用

**2、创建文件夹**

~~~
mkdir -p /Users/lison/work/data/dockerData/seata/resources
~~~

**3、创建临时容器**

~~~shell
docker run -d \
--name seata \
seataio/seata-server
~~~

**4、拷贝容器内置配置文件**

~~~
docker cp seata:/seata-server/resources /Users/lison/work/data/dockerData/seata/
~~~

**5、删除临时容器**

~~~
docker rm -f seata
~~~



### **导入Seata配置到Nacos**

由于我们需要使用nacos作为seata服务的配置中心和注册中心，其中，配置中心的配置，我们需要先行导入

> https://github.com/apache/incubator-seata/tree/2.x/script/config-center

里面有很多配置，但我们只取重点需要的，如下：

~~~properties

transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableClientBatchSendRequest=true
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
service.vgroupMapping.my_test_tx_group=default
service.default.grouplist=127.0.0.1:8091
service.enableDegrade=false
service.disableGlobalTransaction=false
client.rm.asyncCommitBufferLimit=10000
client.rm.lock.retryInterval=10
client.rm.lock.retryTimes=30
client.rm.lock.retryPolicyBranchRollbackOnConflict=true
client.rm.reportRetryCount=5
client.rm.tableMetaCheckEnable=false
client.rm.tableMetaCheckerInterval=60000
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.rm.sagaBranchRegisterEnable=false
client.rm.tccActionInterceptorOrder=-2147482648
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
client.tm.defaultGlobalTransactionTimeout=60000
client.tm.degradeCheck=false
client.tm.degradeCheckAllowTimes=10
client.tm.degradeCheckPeriod=2000
client.tm.interceptorOrder=-2147482648
store.mode=db
store.lock.mode=file
store.session.mode=file
store.file.dir=file_store/data
store.file.maxBranchSessionSize=16384
store.file.maxGlobalSessionSize=512
store.file.fileWriteBufferCacheSize=16384
store.file.flushDiskMode=async
store.file.sessionReloadReadSize=100
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://172.18.0.2:3306/seata-server?useUnicode=true&rewriteBatchedStatements=true
store.db.user=root
store.db.password=123456
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000
store.redis.mode=single
store.redis.single.host=127.0.0.1
store.redis.single.port=6379
store.redis.maxConn=10
store.redis.minConn=1
store.redis.maxTotal=100
store.redis.database=0
store.redis.queryLimit=100
server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
server.distributedLockExpireTime=10000
client.undo.dataValidation=true
client.undo.logSerialization=jackson
client.undo.onlyCareUpdateColumns=true
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.undo.compress.enable=true
client.undo.compress.type=zip
client.undo.compress.threshold=64k
log.exceptionRate=100
transport.serialization=seata
transport.compressor=none
metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898





~~~

将上述配置导入nacos，命名seata-server.properties，如下图:

![image-20240524174544128](typora-user-images/image-20240524174544128.png)





### **修改application.yml配置文件**

在上面从容器中拷贝出来的resources文件夹中找到application.yml文件,根据你实际的nacos等配置信息，设置相应的application.yml配置项

~~~yaml
#  Copyright 1999-2019 Seata.io Group.
#
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#
#  http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.
server:
  port: 7091

spring:
  application:
    name: seata-server
    
console:
  user:
    username: seata
    password: seata

seata:
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: nacos
    nacos:
      server-addr: 172.18.0.6:8848   # nacos的访问地址，因为是在docker中，ip地址改为宿主机地址
      namespace: seata
      group: DEFAULT_GROUP  # nacos的分组
      username: nacos     # nacos的用户名
      password: nacos     # nacos的密码
      context-path:
      #file-extension: yaml # 配置文件格式
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key:
      #secret-key:
      data-id: seata-server.properties # nacos中的配置文件名称
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
    nacos:
      application: seata-server       # seata启动后在nacos的服务名
      server-addr: 172.18.0.6:8848  # nacos的访问地址，因为是在docker中，ip地址改为宿主机地址
      group: DEFAULT_GROUP   # nacos的分组
      namespace: seata
      cluster: default     # 这个歌参数在每个微服务seata时会用到
      username: nacos      # nacos的用户名
      password: nacos      # nacos的密码
      context-path:
      ##if use MSE Nacos with auth, mutex with username/password attribute
      #access-key:
      #secret-key:

  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login    


~~~

如果你想修改到时候生成的web登陆的账号密码，就修改console里的username和password



### 生成seata所需mysql表

TC 运行需要将事务信息保存在数据库，因此需要创建一些表

- https://github.com/apache/incubator-seata/blob/2.x/script/server/db/mysql.sql



访问上面链接，去到源码中，找到script\server\db 这个目录。由于是使用mysql的，所以下载mysql.sql

~~~sql
--
-- Licensed to the Apache Software Foundation (ASF) under one or more
-- contributor license agreements.  See the NOTICE file distributed with
-- this work for additional information regarding copyright ownership.
-- The ASF licenses this file to You under the Apache License, Version 2.0
-- (the "License"); you may not use this file except in compliance with
-- the License.  You may obtain a copy of the License at
--
--     http://www.apache.org/licenses/LICENSE-2.0
--
-- Unless required by applicable law or agreed to in writing, software
-- distributed under the License is distributed on an "AS IS" BASIS,
-- WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
-- See the License for the specific language governing permissions and
-- limitations under the License.
--

-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_status_gmt_modified` (`status` , `gmt_modified`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_status` (`status`),
    KEY `idx_branch_id` (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
    `lock_key`       CHAR(20) NOT NULL,
    `lock_value`     VARCHAR(20) NOT NULL,
    `expire`         BIGINT,
    primary key (`lock_key`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
~~~



### **制作docker-compose.yaml文件**

~~~yaml
# Seata 基础组件服务
version: '3.8'
services:
# seata服务1
  seata-server1:
    image: seataio/seata-server
    hostname: seata-server1
    ports:
      - 8091:8091
      - 7091:7091
    environment:
      - TZ=Asia/Shanghai
      - LANG=en_US.UTF-8
      - STORE_MODE=db
      - SEATA_IP=127.0.0.1
      - SEATA_PORT=8091
    volumes:
      - /Users/lison/work/data/dockerData/seata/resources:/seata-server/resources
~~~

>SEATA_IP：可选，指定seata-server启动的IP，该IP用于向注册中心注册时使用
>SEATA_PORT：可选，指定seata-server启动的端口，默认为 8091
>STORE_MODE：可选，指定seata-server的事务日志存储方式，支持 db、file、redis，默认是 file
>SERVER_NODE：可选, 用于指定seata-server节点ID, 如 1,2,3…，默认根据IP生成
>SEATA_ENV：可选，指定 seata-server 运行环境，如 dev、test 等，服务启动时会使用 registry-dev.conf 这样的配置

### 启动验证



**1、启动Seata服务**

~~~
docker-compose up -d
~~~



**2、查看nacos服务里是否启动成功**

![image-20240524175254196](typora-user-images/image-20240524175254196.png)

**3、查看网站**

>http://127.0.0.1:7091/#/



- 默认账号：seata
- 默认密码：seata
- 如果需要修改就改application.yml里的console配置项



![image-20240524175527451](typora-user-images/image-20240524175527451.png)



## Seata实现分布式事务控制



本示例通过Seata中间件实现分布式事务，模拟电商中的下单和扣库存的过程
我们通过订单微服务执行下单操作，然后由订单微服务调用商品微服务扣除库存

### 配置



1、搭建我们所需要写的demo的库springbootbuild,创建一个名为test的数据库,然后执行以下sql代码:

~~~


DROP TABLE IF EXISTS `test`;
CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `one` varchar(255) DEFAULT NULL,
  `two` varchar(255) DEFAULT NULL,
  `createTime` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4;

CREATE TABLE `t_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_name` varchar(255) DEFAULT NULL COMMENT '用户名称',
  `sex` tinyint(1) DEFAULT '0' COMMENT '性别，0：男，1：女',
  `age` int(3) DEFAULT '0' COMMENT '年龄',
  `address` varchar(255) DEFAULT NULL COMMENT '地址',
  `phone` varchar(11) DEFAULT NULL COMMENT '手机',
  `create_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;

INSERT INTO `t_test` VALUES ('1', '1', '2', '2024-05-27 16:07:34');

INSERT INTO `t_user`  VALUES('1', 'name', '0', '11','地址','18818900000','2024-05-27 16:07:34');

~~~





2、引入依赖

~~~

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>1.4.2</version>
        </dependency>
~~~

1.4.2版本不需要在项目里引入file.conf和registry.conf了。



3、增加数据源配置

~~~yaml
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    master:
      url: jdbc:mysql://127.0.0.1:3306/springbootbuild?useSSL=false&serverTimezone=Asia/Shanghai
      username: root
      password: 123456
      driver-class-name: com.mysql.jdbc.Driver
      initialSize: 5
      minIdle: 5
      maxActive: 20
      maxWait: 60000
      timeBetweenEvictionRunsMillis: 60000
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT user()
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      poolPreparedStatements: true
      connection-properties: druid.stat.mergeSql:true;druid.stat.slowSqlMillis:5000
    
seata:
  registry:
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: "seata"
      group: DEFAULT_GROUP
      application: seata-server
      username: nacos
      password: nacos    
~~~



### 配置seata代理数据源

新增DatabaseConfiguration，Seata的RM通过DataSourceProxy才能在业务代码的失误提交时，通过这个切入点，与TC通讯交互，记录undo_log等

~~~java
package com.lison.springcloudservice.config.base;

import com.alibaba.druid.pool.DruidDataSource;
import io.seata.rm.datasource.DataSourceProxy;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

/**
 * @className: com.lison.springcloudservice.config.base-> DatabaseConfiguration
 * @description:
 * @author: Lison
 * @createDate: 2024-05-27
 */
@Configuration
public class DatabaseConfiguration {

    private final ApplicationContext applicationContext;

    public DatabaseConfiguration(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DruidDataSource ds0() {
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }

    @Primary
    @Bean
    public DataSource dataSource(DruidDataSource ds0)  {
        DataSourceProxy pds0 = new DataSourceProxy(ds0);
        return pds0;
    }
}

~~~



### 启动类修改

注意：需要把spirngboot自带的数据源排除掉，否则出现配置的代理数据源与spirngboot自带的形成循环依赖

~~~
//启动时排除springboot自带的数据源配置类
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
@EnableDiscoveryClient
@EnableFeignClients
public class SpringCloudServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudServiceApplication.class, args);
    }

}
~~~



### 添加undo_log表

该表用来事务回滚，分支事务提交时记录事务相关信息，在分布式事务异常时回滚，分布式事务结束后会删除undo_log的记录。
在spring配置指定的数据库中创建表，每个需要注册到seata server的业务模块都有创建该表，创建语句如下：

~~~

-- ----------------------------

-- Table structure for undo_log

-- ----------------------------

DROP TABLE IF EXISTS `undo_log`;
CREATE TABLE `undo_log` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `branch_id` bigint(20) NOT NULL,
  `xid` varchar(100) NOT NULL,
  `context` varchar(128) NOT NULL,
  `rollback_info` longblob NOT NULL,
  `log_status` int(11) NOT NULL,
  `log_created` datetime NOT NULL,
  `log_modified` datetime NOT NULL,
  `ext` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ux_undo_log` (`xid`,`branch_id`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;

-- ----------------------------

-- Records of undo_log
~~~





这里只简单的演示在一个微服务中，一个分布式事物包含两个分支事物。



### seata使用示例



**Controller**

~~~java
package com.lison.springcloudservice.controller;

import com.lison.springcloudservice.service.ITestService;
import com.lison.springcloudservice.service.TestGlobalTransServiceImpl;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;

/**
 * @className: com.lison.springcloudservice.controller-> TestSeataController
 * @description:
 * @author: Lison
 * @createDate: 2024-05-27
 */

@RestController
@RequestMapping(value = "/test/")
public class TestSeataController {
    @Resource
    private TestGlobalTransServiceImpl testGlobalTransService;
    @GetMapping("/seataTrans")
    public String testSeataTrans() throws Exception {
        testGlobalTransService.testTrans();
        return "success";
    }
}

~~~



**Service**

~~~java

package com.lison.springcloudservice.service;

import io.seata.spring.annotation.GlobalTransactional;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * @className: com.lison.springcloudservice.service-> TestGlobalTransService
 * @description:
 * @author: Lison
 * @createDate: 2024-05-27
 */
@Service
public class TestGlobalTransServiceImpl {
    @Autowired
    private ITestService iTestService;
    //@GlobalTransactional
    public void testTrans() throws Exception {
        // 分支事务添加用户信息
        iTestService.insertUser();
        // 分支事务添加测试
        iTestService.insertTest();
        // 抛出异常，事务回滚
        throw new Exception("test exception");
    }
}

~~~

负责分支事务的服务类，单独拿出来是因为spring事务代理要求事务方法如果和调用方法放在一个类中，代理不生效，具体原因不在赘述。





~~~java
package com.lison.springcloudservice.service.impl;

import com.lison.springcloudservice.mapper.TestMapper;
import com.lison.springcloudservice.mapper.UserMapper;
import com.lison.springcloudservice.service.ITestService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * @className: com.lison.springcloudservice.service.impl-> TestServiceImpl
 * @description:
 * @author: Lison
 * @createDate: 2024-05-27
 */
@Service
public class TestServiceImpl  implements ITestService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private TestMapper testMapper;

    @Transactional(rollbackFor = Exception.class)
    @Override
    public void insertUser() {
        userMapper.insert();
    }

    @Transactional(rollbackFor = Exception.class)
    @Override
    public void insertTest() {
        testMapper.insert();
    }





}

~~~



**Mapper**

~~~java
@Mapper
@Component
public interface TestMapper {
    @Insert("insert into t_test(one,two,createTime) values ( 0,18,now())")
    int insert();
}


@Mapper
@Component
public interface UserMapper{
    @Insert("insert into t_user (user_name,sex,age, create_time) values ('aaaaaaaaa', 0,18,now())")
    int insert();

}

~~~



测试结果：

1、调用：http://localhost:18001/test/seataTrans

2、因为 我们的代码中@GlobalTransactional 被注释。当有异常抛出，所有数据库中插入数据。

~~~
2024-05-27 16:58:57.346 [TID:Ignored_Trace] [http-nio-18001-exec-1] INFO  c.l.s.config.seata.SeataInterceptor -xid in RootContext[null] xid in HttpContext[null]
2024-05-27 16:58:57.504 [TID:Ignored_Trace] [http-nio-18001-exec-1] ERROR o.a.c.c.C.[.[.[.[dispatcherServlet] -Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.Exception: test exception] with root cause
java.lang.Exception: test exception
	at com.lison.springcloudservice.service.TestGlobalTransServiceImpl.testTrans(TestGlobalTransServiceImpl.java:24)
	at com.lison.springcloudservice.controller.TestSeataController.$sw$original$testSeataTrans$6lb26m3(TestSeataController.java:25)
	at com.lison.springcloudservice.controller.TestSeataController.$sw$original$testSeataTrans$6lb26m3$accessor$$sw$ful2f31(TestSeataController.java)
	at com.lison.springcloudservice.controller.TestSeataController$$sw$auxiliary$p4qqk42.call(Unknown Source)
	at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:86)
	at com.lison.springcloudservice.controller.TestSeataController.testSeataTrans(TestSeataController.java)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:190)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:138)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:105)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:879)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:793)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at brave.servlet.TracingFilter.doFilter(TracingFilter.java:68)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at brave.servlet.TracingFilter.doFilter(TracingFilter.java:87)
	at org.springframework.cloud.sleuth.instrument.web.LazyTracingFilter.doFilter(TraceWebServletAutoConfiguration.java:139)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.doFilterInternal(WebMvcMetricsFilter.java:109)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202)
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:541)
	at org.apache.catalina.core.StandardHostValve.$sw$original$invoke$6qrnli3(StandardHostValve.java:139)
	at org.apache.catalina.core.StandardHostValve.$sw$original$invoke$6qrnli3$accessor$$sw$p8ebm33(StandardHostValve.java)
	at org.apache.catalina.core.StandardHostValve$$sw$auxiliary$1213ni1.call(Unknown Source)
	at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:86)
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java)
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:373)
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868)
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1594)
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
	at java.lang.Thread.run(Thread.java:748)
2024-05-27 16:58:57.516 [TID:Ignored_Trace] [http-nio-18001-exec-1] INFO  c.l.s.config.seata.SeataInterceptor -xid in RootContext[null] xid in HttpContext[null]
2024-05-27 16:58:58.508 [TID:N/A] [AsyncReporter{org.springframework.cloud.sleuth.zipkin2.sender.RestTemplateSender@3f169051}] WARN  z.r.AsyncReporter$BoundedAsyncReporter -Spans were dropped due to exceptions. All subsequent errors will be logged at FINE level.
~~~



3、修改代码取消注释，再次调用，数据没有保存成功。

~~~
 */
@Service
public class TestGlobalTransServiceImpl {
    @Autowired
    private ITestService iTestService;
    @GlobalTransactional
    public void testTrans() throws Exception {
        // 分支事务添加用户信息
        iTestService.insertUser();
        // 分支事务添加测试
        iTestService.insertTest();
        // 抛出异常，事务回滚
        throw new Exception("test exception");
    }
}
~~~



4、去掉异常，数据保存成功
