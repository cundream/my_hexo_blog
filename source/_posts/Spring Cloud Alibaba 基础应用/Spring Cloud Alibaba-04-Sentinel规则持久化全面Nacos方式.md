---
title:  Spring Cloud Alibaba-04-Sentinel规则持久化Nacos方式-推荐
date: 2025-04-04 20:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2023.10.03`

# Spring Cloud Alibaba-04-Sentinel规则持久化Nacos方式-推荐

[toc]





## Sentinel-Dashboard 添加规则Nacos双向绑定

**官网文档地址**：https://sentinelguard.io/zh-cn/docs/quick-start.html

思路
从动态规则扩展章节得知，可以通过NacosDataSource作为配置数据源
从Sentinel-Dashboard代码中可以得知，dashboard通过http请求推送配置变更



~~~java
publishRules(entity.getApp(), entity.getIp(), entity.getPort()).get(5000, TimeUnit.MILLISECONDS);

   List<FlowRuleEntity> rules = repository.findAllByMachine(MachineInfo.of(app, ip, port));
        return sentinelApiClient.setFlowRuleOfMachineAsync(app, ip, port, rules);

~~~

所以，只需要在dashboard推送变更后，将配置保存到nacos中且搭配NacosDataSource扩展即可实现配置持久化

从dashboard各个规则的Controller中可以发现，每个Controller都有类似repository的注入



~~~java
    @Autowired
    private RuleRepository<DegradeRuleEntity, Long> repository;
~~~

负责规则的增删改查，默认提供的都是内存控制，例如InMemDegradeRuleStore ，每个内存控制类都继承同一个抽象类

~~~java
public class InMemDegradeRuleStore extends InMemoryRuleRepositoryAdapter<DegradeRuleEntity>
~~~



只要在respository写操作后添加配置推送的nacos的操作



## 实现

### 注释掉test

~~~xml
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
<!--            <scope>test</scope>-->
        </dependency>

~~~



### 增加 NacosConfig配置



~~~java
package com.alibaba.csp.sentinel.dashboard.config;


import com.alibaba.nacos.api.NacosFactory;
import com.alibaba.nacos.api.PropertyKeyConst;
import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.exception.NacosException;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.StringUtils;

import java.util.Properties;

/**
 * @className: com.alibaba.csp.sentinel.dashboard.config-> NacosConfig
 * @description:
 * @author: Lison
 * @createDate: 2023-10-03
 */
@Configuration
@EnableConfigurationProperties(NacosConfig.NacosProperties.class)
public class NacosConfig {

    @ConfigurationProperties(prefix = "nacos.config")
    public static class NacosProperties{
        private String group = "DEFAULT_GROUP";
        private String namespace;
        private String authDataId = "sentinel-dashboard-auth";

        private String username;


        private String password;
        private Long timeout = 3000L;

        private String serverAddr;

        public String getGroup() {
            return group;
        }

        public void setGroup(String group) {
            this.group = group;
        }

        public String getNamespace() {
            return namespace;
        }

        public void setNamespace(String namespace) {
            this.namespace = namespace;
        }

        public Long getTimeout() {
            return timeout;
        }

        public void setTimeout(Long timeout) {
            this.timeout = timeout;
        }

        public String getServerAddr() {
            return serverAddr;
        }

        public void setServerAddr(String serverAddr) {
            this.serverAddr = serverAddr;
        }

        public String getAuthDataId() {
            return authDataId;
        }

        public void setAuthDataId(String authDataId) {
            this.authDataId = authDataId;
        }

        public String getUsername() { return username;}

        public void setUsername(String username) { this.username = username;}

        public String getPassword() {return password;}

        public void setPassword(String password) { this.password = password;}
    }

    @Bean
    public ConfigService configService(NacosProperties nacosProperties) throws NacosException {
        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.SERVER_ADDR, nacosProperties.getServerAddr());
        if (!StringUtils.isEmpty(nacosProperties.getNamespace())) {
            properties.setProperty(PropertyKeyConst.NAMESPACE, nacosProperties.getNamespace());
        }

        if (!StringUtils.isEmpty(nacosProperties.getUsername())) {
            properties.setProperty(PropertyKeyConst.USERNAME, nacosProperties.getUsername());
        }

        if (!StringUtils.isEmpty(nacosProperties.getPassword())) {
            properties.setProperty(PropertyKeyConst.PASSWORD, nacosProperties.getPassword());
        }

        return NacosFactory.createConfigService(properties);
    }

}


~~~

### 加入Repository配置

~~~java
package com.alibaba.csp.sentinel.dashboard.config;


import com.alibaba.csp.sentinel.dashboard.datasource.entity.gateway.ApiDefinitionEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.gateway.GatewayFlowRuleEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.AuthorityRuleEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.DegradeRuleEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.FlowRuleEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.ParamFlowRuleEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.RuleEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.SystemRuleEntity;
import com.alibaba.csp.sentinel.dashboard.repository.gateway.InMemApiDefinitionStore;
import com.alibaba.csp.sentinel.dashboard.repository.gateway.InMemGatewayFlowRuleStore;
import com.alibaba.csp.sentinel.dashboard.repository.rule.InMemAuthorityRuleStore;
import com.alibaba.csp.sentinel.dashboard.repository.rule.InMemDegradeRuleStore;
import com.alibaba.csp.sentinel.dashboard.repository.rule.InMemFlowRuleStore;
import com.alibaba.csp.sentinel.dashboard.repository.rule.InMemParamFlowRuleStore;
import com.alibaba.csp.sentinel.dashboard.repository.rule.InMemSystemRuleStore;
import com.alibaba.csp.sentinel.dashboard.repository.rule.InMemoryRuleRepositoryAdapter;
import com.alibaba.csp.sentinel.dashboard.repository.rule.NacosRepository;
import com.alibaba.nacos.api.config.ConfigService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

/**
 * @className: com.alibaba.csp.sentinel.dashboard.config-> NacosRepositoryConfig
 * @description:
 * @author: Lison
 * @createDate: 2023-10-03
 */
@Configuration
public class NacosRepositoryConfig {
    @Autowired
    private ConfigService configService;
    @Autowired
    private NacosConfig.NacosProperties nacosProperties;

    @Bean
    public NacosRepository<ApiDefinitionEntity> apiDefinitionEntityNacosRepository(InMemApiDefinitionStore inMemApiDefinitionStore){
        return buildNacosRepository("gateway-api-definition",inMemApiDefinitionStore);
    };

    @Bean
    public NacosRepository<GatewayFlowRuleEntity> gatewayFlowRuleEntityNacosRepository(InMemGatewayFlowRuleStore inMemGatewayFlowRuleStore){
        return buildNacosRepository("gateway-flow-rule",inMemGatewayFlowRuleStore);
    };

    @Bean
    @Primary
    public NacosRepository<AuthorityRuleEntity> authorityRuleEntityNacosRepository(InMemAuthorityRuleStore inMemAuthorityRuleStore){
        return buildNacosRepository("authority-rule",inMemAuthorityRuleStore);
    };

    @Bean
    @Primary
    public NacosRepository<DegradeRuleEntity> degradeRuleEntityNacosRepository(InMemDegradeRuleStore inMemDegradeRuleStore){
        return buildNacosRepository("degrade-rule",inMemDegradeRuleStore);
    };

    @Bean
    @Primary
    public NacosRepository<FlowRuleEntity> flowRuleEntityNacosRepository(InMemFlowRuleStore inMemFlowRuleStore){
        return buildNacosRepository("flow-rule",inMemFlowRuleStore);
    };

    @Bean
    @Primary
    public NacosRepository<ParamFlowRuleEntity> paramFlowRuleEntityNacosRepository(InMemParamFlowRuleStore inMemParamFlowRuleStore){
        return buildNacosRepository("param-flow-rule",inMemParamFlowRuleStore);
    };

    @Bean
    @Primary
    public NacosRepository<SystemRuleEntity> systemRuleEntityNacosRepository(InMemSystemRuleStore inMemSystemRuleStore){
        return buildNacosRepository("system-rule",inMemSystemRuleStore);
    };

    private <T extends RuleEntity> NacosRepository<T> buildNacosRepository(String suffix, InMemoryRuleRepositoryAdapter<T> ruleRepositoryAdapter){
        NacosRepository<T> nacosRepository = new NacosRepository<>(suffix);
        nacosRepository.setNacosProperties(nacosProperties);
        nacosRepository.setRuleRepositoryAdapter(ruleRepositoryAdapter);
        nacosRepository.setConfigService(configService);
        return nacosRepository;
    }

}

~~~



~~~java
package com.alibaba.csp.sentinel.dashboard.datasource.entity;

import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.RuleEntity;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.util.Map;

/**
 * @className: com.alibaba.csp.sentinel.dashboard.datasource.entity-> NacosEntity
 * @description:
 * @author: Lison
 * @createDate: 2023-10-03
 */public class NacosEntity<T extends RuleEntity> {

    private T rule;

    public NacosEntity(T sentinelRule) {
        this.rule = sentinelRule;
    }

    // Getter方法
    public T getRule() {
        return rule;
    }

}

~~~



### 注入Repository

~~~java
package com.alibaba.csp.sentinel.dashboard.repository.rule;


import com.alibaba.csp.sentinel.dashboard.config.NacosConfig;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.NacosEntity;
import com.alibaba.csp.sentinel.dashboard.datasource.entity.rule.RuleEntity;
import com.alibaba.csp.sentinel.dashboard.discovery.MachineInfo;
import com.alibaba.fastjson.JSON;
import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.config.ConfigType;

import java.util.List;
import java.util.stream.Collectors;

/**
 * @className: com.alibaba.csp.sentinel.dashboard.repository.rule-> NacosRepository
 * @description:
 * @author: Lison
 * @createDate: 2023-10-03
 */

public class NacosRepository<T extends RuleEntity> implements RuleRepository<T, Long> {

    private ConfigService configService;

    private NacosConfig.NacosProperties nacosProperties;

    private InMemoryRuleRepositoryAdapter<T> ruleRepositoryAdapter;

    private final String suffix;

    public NacosRepository(String suffix){
        this.suffix = suffix;
    }

    public ConfigService getConfigService() {
        return configService;
    }

    public void setConfigService(ConfigService configService) {
        this.configService = configService;
    }

    public NacosConfig.NacosProperties getNacosProperties() {
        return nacosProperties;
    }

    public void setNacosProperties(NacosConfig.NacosProperties nacosProperties) {
        this.nacosProperties = nacosProperties;
    }

    public InMemoryRuleRepositoryAdapter<T> getRuleRepositoryAdapter() {
        return ruleRepositoryAdapter;
    }

    public void setRuleRepositoryAdapter(InMemoryRuleRepositoryAdapter<T> ruleRepositoryAdapter) {
        this.ruleRepositoryAdapter = ruleRepositoryAdapter;
    }

    @Override
    public T save(T entity) {
        T save = ruleRepositoryAdapter.save(entity);
        publishConfig(entity.getApp());
        return save;
    }

    @Override
    public List<T> saveAll(List<T> rules) {
        List<T> ts = ruleRepositoryAdapter.saveAll(rules);
        if(ts!=null && !ts.isEmpty()){
            publishConfig(ts.get(0).getApp());
        }
        return ts;
    }

    @Override
    public T delete(Long id) {
        T delete = ruleRepositoryAdapter.delete(id);
        publishConfig(delete.getApp());
        return delete;
    }

    @Override
    public T findById(Long id) {
        return ruleRepositoryAdapter.findById(id);
    }

    @Override
    public List<T> findAllByMachine(MachineInfo machineInfo) {
        return ruleRepositoryAdapter.findAllByMachine(machineInfo);
    }

    @Override
    public List<T> findAllByApp(String appName) {
        return ruleRepositoryAdapter.findAllByApp(appName);
    }

    private void publishConfig(String app) {
        try {
            List<T> allByApp = ruleRepositoryAdapter.findAllByApp(app);
            List<Object> rules = allByApp.stream().map(NacosEntity::new).map(NacosEntity::getRule).collect(Collectors.toList());
            configService.publishConfig(app + "-" + suffix, nacosProperties.getGroup(), JSON.toJSONString(rules), ConfigType.JSON.getType());
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

}


~~~





### 最后替换掉对应Controller中对应的repository，

~~~java
@RestController
@RequestMapping(value = "/v1/flow")
public class FlowControllerV1 {

    private final Logger logger = LoggerFactory.getLogger(FlowControllerV1.class);

    @Autowired
    private NacosRepository<FlowRuleEntity> repository;
}
~~~

### 添加参数配置

~~~properties

nacos.config.group=DEFAULT_GROUP;
nacos.config.namespace=sentinelId;
nacos.config.password=nacos;
nacos.config.serverAddr=http://127.0.0.1:8848;
nacos.config.username=nacos

~~~

客户端需要添加pom

~~~xml
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
            <version>xxxx</version>
        </dependency>

~~~



![image-20240221155333042](typora-user-images/image-20240221155333042.png)