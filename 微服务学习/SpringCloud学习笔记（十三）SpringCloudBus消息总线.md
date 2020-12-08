# SpringCloud Bus消息总线

## 本片要点

- 简要介绍总线的概念，以及分布式系统解决的问题。
- 介绍Docker安装RabbitMQ的基本命令。
- SpringCloud Bus+ RabbitMQ实现全局动态刷新。

## Spring Cloud Bus简介

> https://spring.io/projects/spring-cloud-bus

### 概述

SpringCloud Bus是将分布式系统的节点与轻量级消息系统链接起来的框架，它整合了Java的事件处理机制和消息中间件的功能。目前支持RabbitMQ和Kafka。【本篇文章使用RabbitMQ作为消息代理】

管理和传播分布式系统间的消息，像一个分布式执行器，用于广播状态更改，事件推送等，也可以作为微服务间的通信通道。

SpringCloud Bus可以配合SpringCloud Config实现配置的动态刷新。

### 什么是总线

在微服务架构的系统中，通常会使用**轻量级的消息代理**来构建一个**共用的消息主题**，并让系统中所有微服务实例都连接上来。由于**该主题中产生的消息会被所有实例监听和消费，所以称之为消息总线**。总线上的各个实例，都可以方便地广播一些需要让其他连接在该主题上的实例都知道的消息。

### 基本原理

ConfigClient 实例都监听MQ中同一个topic【默认是SpringCloudBus】。当一个服务刷新数据的时候，它会把这个消息放入Topic中，这样其他监听同一Topic的服务就能够得到通知，然后去更新自身的配置。

## Docker安装RabbitMQ

> 可以参考：[SpringBoot整合RabbitMQ以及Rabbit队列学习](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/106958412#LinuxRabbitmq_60)

```shell
$ docker pull rabbitmq:3-management # management带web界面管理
$ docker images  # 查看image ID

$ docker run -d --name myrabbit -p 5672:5672 -p 15672:15672 cc86ffa2f398 #启动  最后跟着 image ID

$ systemctl status firewalld #查看防火墙的状态【(running)意思是打开，我们需要设置开放的端口】
$ firewall-cmd --list-ports #查看防火墙开放的端口
$ firewall-cmd --zone=public --add-port=15672/tcp --permanent # 开放15672,15672是Web管理界面的端口
$ firewall-cmd --zone=public --add-port=5672/tcp --permanent # 开放5672,5672是MQ访问的端口
$ firewall-cmd --reload # 使修改生效
```

访问15672端口即可进入RabbitMQ的web管理界面，账号密码默认都是guest。

![image-20201129130431595](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%89%EF%BC%89SpringCloudBus%E6%B6%88%E6%81%AF%E6%80%BB%E7%BA%BF/image-20201129130431595.png)

## 演示动态刷新全局广播前置准备

> 前提条件：RabbitMQ环境已成功安装。

接下来，仿照3355结构，再新建一个3366。

### 新建模块，引入依赖

```xml
        <!--添加消息总线RabbitMQ支持-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>	
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

### 配置bootstrap.yml

```yml
server:
  port: 3366

spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称   
      uri: http://localhost:3344 #配置中心地址

  #rabbitmq相关配置 15672是Web管理界面的端口；5672是MQ访问的端口
  rabbitmq:
    host: [your hostname]
    port: 5672
    username: guest
    password: guest

#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka

# 暴露监控端点
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### 编写主启动类

```java
@EnableEurekaClient
@SpringBootApplication
public class ConfigClientMain3366 {
    public static void main(String[] args) {
        SpringApplication.run(ConfigClientMain3366.class, args);
    }
}
```

### 编写接口

```java
@RestController
@RefreshScope
public class ConfigClientController {
    @Value("${server.port}")
    private String serverPort;

    @Value("${config.info}")
    private String configInfo;

    @GetMapping("/configInfo")
    public String configInfo() {
        return "serverPort: " + serverPort + "\t\n\n configInfo: " + configInfo;
    }
}
```

## 设计思想

设计思想主要是以下两种：

一、利用消息总线触发一个客户端`/bus/refresh`，而刷新所有客户端的配置。

![image-20201129133230715](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%89%EF%BC%89SpringCloudBus%E6%B6%88%E6%81%AF%E6%80%BB%E7%BA%BF/image-20201129133230715.png)

二、利用消息总线触发一个服务端ConfigServer的`/bus/refresh`，而刷新所有客户端的配置。

![image-20201129133500215](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%89%EF%BC%89SpringCloudBus%E6%B6%88%E6%81%AF%E6%80%BB%E7%BA%BF/image-20201129133500215.png)

相比之下，图二的架构【通知服务端ConfigServer】更加合理，图一不合理的原因如下：

1. 打破了微服务的职责单一性，微服务本身为业务模块，不应该担任刷新配置的职责。
2. 破坏了微服务各节点的对等性。
3. 存在局限，如微服务迁移时，网络地址时常发生变化，这时如果希望自动刷新，会作更多的修改。

## 开始演示动态刷新全局广播

> 前提：前置准备已经完成，此时拥有3344ConfigServer，3355和3366Client，和Eureka配置中心7001。

### 为三个模块都添加消息总线支持

```xml
        <!--添加消息总线RabbitMQ支持-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
        </dependency>
```

### 为三个模块配置yml

```yml
#rabbitmq相关配置
  rabbitmq:
    host: [your hostname]
    port: 5672
    username: guest
    password: guest
```

### 为ConfigServer配置yml

```yml
#rabbitmq相关配置,暴露bus刷新配置的端点
management:
  endpoints: #暴露bus刷新配置的端点
    web:
      exposure:
        include: 'bus-refresh'
```

### 为ConfigClient配置yml

```yml
# 暴露监控端点
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### 测试

依次启动7001，3344，3355，3366模块，进行测试，依次访问：

- http://localhost:3344/master/config-dev.yml
- http://localhost:3355/configInfo
- http://localhost:3366/configInfo

暂时是没有任何问题的，配置信息能够成功从Gitee上获取得到。

此时改变配置信息的版本号，并向ConfigServer发送一次POST请求：

```shell
$ curl -X POST "http://localhost:3344/actuator/bus-refresh"
```

再次测试3355和3366config client，已经成功实现一次通知，处处更新。

### 原理回顾

ConfigClient 实例都监听RabbitMQ中同一个topic【默认是SpringCloudBus】。当一个服务刷新数据的时候，它会把这个消息放入Topic中，这样其他监听同一Topic的服务就能够得到通知，然后去更新自身的配置。

![image-20201129140404201](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%89%EF%BC%89SpringCloudBus%E6%B6%88%E6%81%AF%E6%80%BB%E7%BA%BF/image-20201129140404201.png)

## 动态刷新定点通知

我们已经通过Spring Cloud Bus + RabbitMQ实现了一处通知，全局广播，处处更新。那，如果我只想通知其中某个client更新呢？如何定点通知只指定某个实例生效呢？

我们可以通过通知的url指定实例的destination：`http://localhost:3344/actuator/bus-refresh/{destination}`，此时`bus/refresh`通知会通过destination参数类指定需要更新配置的服务或实例。

比如，只通知3355可以发送下面这个请求：

```shell
$ curl -X POST "http://localhost:3344/actuator/bus-refresh/config-client:3355"
```

- config-client为spring.application.name。
- 3355为对应的端口号。

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。