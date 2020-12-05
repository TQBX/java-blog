## 本篇要点

- 简单了解Nacos提供的功能。
- 简单介绍Nacos安装配置及启动。
- 演示Nacos作为服务注册中心的用法。
- Nacos与其他注册中心的对比。

## Nacos简介

> **Nacos** = (Dynamic) **Na**ming and **Co**nfiguration **S**ervice 注册中心+配置中心，也就是代替Eureka作为服务注册中心，替代Config作为配置中心，替代Bus作为消息总线。
>
> 官方网站: http://nacos.io

Nacos是一个更易于构建云原生应用的**动态服务发现、配置管理和服务管理**平台。

服务是Nacos中的头等公民。Nacos支持几乎所有类型的服务，入：Dubbo/gRPC，Spring Cloud RESTFUL服务或Kubernetes服务。

## Nacos主要提供的四种功能

### 服务发现和服务运行状况检查

Nacos使服务易于注册自己并通过DNS或HTTP接口发现其他服务。 Nacos还提供服务的实时运行状况检查，以防止向不正常的主机或服务实例发送请求。

### 动态配置管理

动态配置服务使您可以在所有环境中以集中和动态的方式管理所有服务的配置。 Nacos消除了在更新配置时重新部署应用程序和服务的需求，这使配置更改更加有效和敏捷。

### 动态DNS服务

Nacos支持加权路由，使您可以更轻松地在数据中心内的生产环境中实施中间层负载平衡，灵活的路由策略，流控制和简单的DNS解析服务。它可以帮助您轻松实现基于DNS的服务发现，并防止应用程序耦合到特定于供应商的服务发现API。

### 服务和元数据管理

Nacos提供了易于使用的服务仪表板，可帮助您管理服务元数据，配置，kubernetes DNS，服务运行状况和指标统计信息。

## Windows中Nacos下载及安装

> 推荐下载稳定版本：Nacos1.3.1

下载地址：[https://github.com/alibaba/nacos/releases/tag/1.3.1](https://github.com/alibaba/nacos/releases/tag/1.3.1)

下载之后，进入bin目录，`cmd startup.cmd -m standalone`启动单机模式。

接着访问：[http://localhost:8848/nacos/](http://localhost:8848/nacos/)，账号密码都是`nacos`。

## 作为服务注册中心演示

### 新建服务模块

新建模块：`cloudalibaba-provider-payment9001`，引入依赖：

```xml
        <!--SpringCloud ailibaba nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```

### 编写yml配置

```yml
server:
  port: 9001

spring:
  application:
    name: nacos-payment-provider
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #配置Nacos地址

management:
  endpoints:
    web:
      exposure:
        include: '*'
```

### 主启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class PaymentMain9001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentMain9001.class, args);
    }
}
```

### Controller接口

```java
@RestController
public class PaymentController {
    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/nacos/{id}")
    public String getPayment(@PathVariable("id") Integer id) {
        return "nacos registry, serverPort: " + serverPort + "\t id" + id;
    }
}
```

### 测试

启动nacos，启动9001服务，访问`localhost:8848/nacos`。

![image-20201205215731911](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E5%8F%91%E7%8E%B0%E9%85%8D%E7%BD%AE/image-20201205215731911.png)

服务已经成功注册进nacos注册中心。

## 演示负载均衡

仿照9001模块再建一个9002模块，具体步骤就省略了，端口号改一改就可以。接着依次启动nacos，9001，9002，观察nacos服务注册中心的情况：

![image-20201205220849413](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E5%8F%91%E7%8E%B0%E9%85%8D%E7%BD%AE/image-20201205220849413.png)

`nacos-payment-provider`服务下包含了两个实例。

### 新建消费者模块

新建`cloudalibaba-consumer-nacos-order83`，依旧引入依赖：

```xml
        <!--SpringCloud ailibaba nacos -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```

> nacos默认支持负载均衡：是因为该依赖已经集成ribbon，故天然支持。

### 编写yml配置

```yml
server:
  port: 83


spring:
  application:
    name: nacos-order-consumer
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848


#消费者将要去访问的微服务名称(注册成功进nacos的微服务提供者)
service-url:
  nacos-user-service: http://nacos-payment-provider
```

### 主启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class OrderNacosMain83 {
    public static void main(String[] args) {
        SpringApplication.run(OrderNacosMain83.class, args);
    }
}
```

### Controller接口

```java
@RestController
@Slf4j
public class OrderNacosController {
    @Resource
    private RestTemplate restTemplate;

    @Value("${service-url.nacos-user-service}")
    private String serverURL;

    @GetMapping(value = "/consumer/payment/nacos/{id}")
    public String paymentInfo(@PathVariable("id") Long id) {
        return restTemplate.getForObject(serverURL + "/payment/nacos/" + id, String.class);
    }

}
```

### 配置类

```java
@Configuration
public class ApplicationContextConfig {
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}
```

### 测试一下

启动nacos，80消费者，9001，9002服务提供模块。

访问：`http://localhost:83/consumer/payment/nacos/1`，将会轮询访问服务接口。

## Nacos与其他注册中心对比

![image-20201205235848405](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E5%8F%91%E7%8E%B0%E9%85%8D%E7%BD%AE/image-20201205235848405.png)

## [源码下载](https://tqbx.gitee.io/javablog/#/微服务学习/SpringCloud学习笔记（二）Eureka服务注册与发现?id=源码下载)

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。