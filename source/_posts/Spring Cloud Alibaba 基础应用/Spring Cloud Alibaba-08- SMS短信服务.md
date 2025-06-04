---
title: Spring Cloud Alibaba-08-SMS短信服务
date: 2025-04-08 19:10:56
---


`Lison `  `<dreamlison@163.com>`,  `v1.0.0`, `2024.5.1`

# Spring Cloud Alibaba-08-SMS短信服务

## 短信服务介绍

> 短信服务(Short Message Service)是阿里云为用户提供的一种通信服务的能力。
>
> 产品优势:覆盖全面、高并发处理、消息堆积处理、开发管理简单、智能监控调度
> 产品功能:短信通知、短信验证码、推广短信、异步通知、数据统计
> 应用场景:短信验证码、系统信息推送、推广短信等

![image-20240510121326830](typora-user-images/image-20240510121326830.png)



## 短信服务使用





1、入驻阿里云

![image-20240510121854650](typora-user-images/image-20240510121854650.png)



2、开通短信服务，按流程创建资质、申请前面，创建模版，系统设置，发送短信

![image-20240510121631586](typora-user-images/image-20240510121631586.png)





## SMS概念

- 短信服务（Short Message Service）

  短信服务是广大企业客户快速触达手机用户所优选使用的通信能力。调用API或用群发助手，即可发送验证码、通知类和营销类短信。

- 短信模版（TemplateId）

  使用短信服务首先都需要创建短信模板提交审核，这样可以防止不法分子通过云服务商提供的短信服务实施短信诈骗。

- 短信签名（SignName）

  短信末尾会附上签名以识别此条短信是由谁发送，这样可以令用户对短信来源有一个明确的印象。

- 地域（RegionId）

  地域表示SMS的数据中心所在物理位置。可以根据费用、请求来源等选择合适的地域，一般是阿里云短信配置。

- 访问密钥（AccessKey）

  AccessKey简称AK，指的是访问身份验证中用到的AccessKey ID和AccessKey Secret。SMS通过使用AccessKey ID和AccessKey Secret对称加密的方法来验证某个请求的发送者身份。AccessKey ID用于标识用户；AccessKey Secret是用户用于加密签名字符串和SMS用来验证签名字符串的密钥，必须保密。关于获取AccessKey的方法







目前BladeX提供的blade-starter-sms集成了四种sms，分别为：云片sms、阿里云sms、七牛sms、腾讯sms



## 功能测试

引入依赖:

~~~xml
<!--短信发送-->
 <dependency> 
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alicloud-sms</artifactId>
</dependency>

~~~

~~~yaml
sms:
  enabled: true
  name: aliyun
  template-id: SMS_xxxx
  sign-name: xxxxx
  access-key: xxxxxxx
  secret-key: xxxxxxx
  region-id: cn-hangzhou
~~~





使用阿里云提供的Demo测试短信发送

~~~java
public class SmsUtil {
    //替换成自己申请的accessKeyId
    private static String accessKeyId = "xxx"; //替换成自己申请的accessKeySecret
    private static String accessKeySecret = "xxxxx";
    static final String product = "Dysmsapi";
    static final String domain = "dysmsapi.aliyuncs.com";

    /**
     * 发送短信
     *
     * @param phoneNumbers 要发送短信到哪个手机号
     * @param signName     短信签名[必须使用前面申请的]
     * @param templateCode 短信短信模板ID[必须使用前面申请的]
     * @param param        模板中${code}位置传递的内容
     */
    public static void sendSms(String phoneNumbers, String signName, String templateCode, String param) {
        try {
            System.setProperty("sun.net.client.defaultConnectTimeout", "10000");
            System.setProperty("sun.net.client.defaultReadTimeout", "10000");
            //初始化acsClient,暂不支持region化
            IClientProfile profile = DefaultProfile.getProfile("cn-hangzhou", accessKeyId, accessKeySecret);

            DefaultProfile.addEndpoint("cn-hangzhou", "cn-hangzhou", product, domain);
            IAcsClient acsClient = new DefaultAcsClient(profile);
            SendSmsRequest request = new SendSmsRequest();
            request.setPhoneNumbers(phoneNumbers);
            request.setSignName(signName);
            request.setTemplateCode(templateCode);
            request.setTemplateParam(param);
            request.setOutId("yourOutId");
            SendSmsResponse sendSmsResponse = acsClient.getAcsResponse(request);
            if (!"OK".

                    equals(sendSmsResponse.getCode())) {
                throw new RuntimeException(sendSmsResponse.getMessage());
            }
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("发送短信失败");
        }
    }
}


~~~

