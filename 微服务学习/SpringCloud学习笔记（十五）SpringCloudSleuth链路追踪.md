# Spring Cloud Sleuth链路追踪

> https://spring.io/projects/spring-cloud-sleuth#overview

## 本篇要点

- 简单介绍Spring Cloud Sleuth。
- 介绍zipkin环境搭建。
- 演示链路追踪效果。

## 分布式服务追踪与调用链系统产生的背景

在微服务框架中，一个由客户端发起的请求在后端系统中会经过多个不同的服务节点调用来协同产生最后的请求结果，每一个前端请求都会形成一条**复杂的分布式服务调用链路**，链路中的任何一环出现高延时或错误都会引起整个请求最后的失败。

## Spring Cloud Sleuth是什么？

Spring Cloud Sleuth为Spring Cloud实现了一种分布式跟踪解决方案。

完整的调用链路：一条链路通过Trace Id唯一标识，Span标识发起的请求信息，各span通过parent id关联起来。

![trace-id](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%BA%94%EF%BC%89SpringCloudSleuth%E9%93%BE%E8%B7%AF%E8%BF%BD%E8%B8%AA/trace-id.png)

## Zipkin环境搭建

> 下载地址：https://dl.bintray.com/openzipkin/maven/io/zipkin/java/zipkin-server/

运行zipkin：

```shell
$ java -jar zipkin-server-2.12.9-exec.jar
```

访问：`localhost:9411/zipkin/`

![image-20201203235928233](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%BA%94%EF%BC%89SpringCloudSleuth%E9%93%BE%E8%B7%AF%E8%BF%BD%E8%B8%AA/image-20201203235928233.png)

## 链路追踪演示

本次链路追踪的演示，选择不新建项目，选取老项目演示。

选取`cloud-provider-payment8001`作为服务的提供者，选取`cloud-consumer-order80`作为服务的消费者。

### 服务提供者和消费者配置

俩服务都添加配置pom.xml

```xml
        <!--包含了sleuth+zipkin-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```

俩服务都添加配置yml

```yml
spring:
  application:
    name: cloud-payment-service # 服务名称!
  zipkin:
    base-url: http://localhost:9411 # 监控中心
  sleuth:
    sampler:
    #采样率值介于 0 到 1 之间，1 则表示全部采集
    probability: 1
```

服务提供者提供测试接口

```java
    @GetMapping("/payment/zipkin")
    public String paymentZipkin(){
        return "payment zipkin !";
    }
```

服务消费者提供测试接口

```java
    @GetMapping("/consumer/payment/zipkin")
    public String paymentZipkin(){
        return restTemplate.getForObject("http://localhost:8001" + "/payment/zipkin/",String.class);
    }
```

### 测试

依次启动7001，80，8001模块，消费者调用测试接口，产生调用链路，进入zipkin管理页面，可以观测调用链路的响应情况等。

![image-20201204215550223](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%BA%94%EF%BC%89SpringCloudSleuth%E9%93%BE%E8%B7%AF%E8%BF%BD%E8%B8%AA/image-20201204215550223.png)

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。