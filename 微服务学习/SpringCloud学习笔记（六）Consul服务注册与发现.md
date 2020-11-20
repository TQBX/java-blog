## Consul简介

> https://www.consul.io/docs/intro

Consul是一套开源的分布式服务发现和配置管理系统，由HashiCorp公司用go语言开发。

提供了微服务系统中的服务治理、配置中心、控制总线等功能，他们可以单独使用，也可一起使用构建全方位的服务网格。 总之，Consul提供了一种完整的服务网格解决方案。

关键特性：

1. **Service Discovery**服务发现
2. **Health Checking**健康检查
3. **KV Store**键值对存储
4. **Secure Service Communication**安全的服务通信
5. **Multi**多数据中心

## Consul安装与使用

### Windows安装

进入下载界面：https://www.consul.io/downloads

在`consul.exe`同级目录中打开控制台，执行：

```shell
consul --version #查看版本
consul agent -dev #开发者模式启动
```



![image-20201120225959604](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%85%AD%EF%BC%89Consul%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120225959604.png)

### Docker安装

直接看这篇文章就好了：[Docker安装Consul](https://www.cnblogs.com/summerday152/p/14013439.html)

## 注册服务提供者

### 引入依赖

```xml
<!--SpringCloud consul-server -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

### 配置yml

```yml

#consul服务端口号
server:
  port: 8006

spring:
  application:
    name: consul-provider-payment
  #consul注册中心地址
  cloud:
    consul:
      host: 127.0.0.1
      port: 8500
      discovery:
        #hostname: 127.0.0.1
        service-name: ${spring.application.name}
```

### 添加注解

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentConsul8006Application {
    public static void main(String[] args) {
        SpringApplication.run(PaymentConsul8006Application.class, args);
    }
}
```

### 编写Controller

```java
@RestController
@Slf4j
public class PaymentController {
    @Value("${server.port}")
    private String serverPort;

    @GetMapping(value = "/payment/consul")
    public String paymentConsul() {
        return "spring cloud with consul: " + serverPort + "\t   " + UUID.randomUUID().toString();
    }
}
```

同样的，我们希望将这个服务注册进Consul服务中心。

### 测试

我们启动`PaymentConsul8006Application`，再查看`localhost:8500/`：

![image-20201120232400144](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%85%AD%EF%BC%89Consul%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120232400144.png)

## 注册服务消费者

pom，yml，启动类注解差不太多，这里就不赘述了，感兴趣可以查看仓库代码：https://gitee.com/tqbx/spring-cloud-learning，以标签的形式详细区分每个步骤。

当我们同时启动消费者和提供者，Consul中就会注册进两个service。消费者访问`localhost/consumer/payment/consul`，将会调用提供者的接口，完成需求。

![image-20201120233424931](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%85%AD%EF%BC%89Consul%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120233424931.png)

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。