# Spring Cloud Stream消息驱动

> - [Introducing Spring Cloud Stream](https://docs.spring.io/spring-cloud-stream/docs/3.0.10.RELEASE/reference/html/spring-cloud-stream.html#spring-cloud-stream-overview-introducing)
> - https://spring.io/projects/spring-cloud-stream

## 本篇要点

- 简单介绍Spring Cloud Stream及其作用。
- 演示消息驱动的过程。
- 演示分组消费和chi'jiu'hua

## Spring Cloud Stream概述

> Spring Cloud Stream is a framework for building highly scalable event-driven microservices connected with shared messaging systems.
>
> The framework provides a flexible programming model built on already established and familiar Spring idioms and best practices, including support for persistent pub/sub semantics, consumer groups, and stateful partitions.

Spring Cloud Stream是一个用于构建**与共享消息传递系统连接的高度可扩展的事件驱动型微服务**的框架。

应用程序通过inputs或outputs来与Spring Cloud Stream中binder对象交互，binder对象负责与消息中间件交互。也就是说：**Spring Cloud Stream能够屏蔽底层消息中间件【RabbitMQ，kafka等】的差异，降低切换成本，统一消息的编程模型**。因此，如果我们想要使用消息驱动，我们不需要了解各种消息中间件，我们只需要了解Spring Cloud Stream就好了。

SpringCloud Stream通过Spring Integration来连接消息代理中间件以实现消息事件驱动。

SpringCloud Stream还为一些供应商的消息中间件产品提供了个性化的自动化配置实现，引用了发布-订阅，消费组，分区三个核心概念。

目前支持的消息中间件官网可以查询：像RabbitMQ，kafka都是支持的，本篇文章基于RabbitMQ消息中间件。

## 设计思想

### 标准的MQ

![image-20201203001248009](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/image-20201203001248009.png)

标准消息队列的特点：

1. 生产者/消费者之间靠消息媒介传递消息内容Message。
2. 消息必须走特定的通道MessageChannel。
3. 消息通道子接口SubscribableChannel，由MessageHandler消息处理器所订阅。

### Spring Cloud Stream

![SCSt-overview](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/SCSt-overview.png)

如何统一底层差异？**通过定义binder绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离**。通过向应用程序暴露统一的Channel通道，使得应用程序不再需要考虑各种不同的消息中间件的实现。

Stream中的消息通信方式遵循**发布-订阅模式**，使用Topic主题进行广播。

![SCSt-with-binder](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/SCSt-with-binder.png)

### API及常用注解

| 组成            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| Middleware      | 中间件，目前只支持RabbitMQ和Kafka                            |
| Binder          | Binder是应用与消息中间件之间的封装，目前实行了Kafka和RabbitMQ的Binder，<br />通过Binder可以很方便连接中间件，可以动态地改变消息类型 |
| @Input          | 注解标识输入通道，通过该输出通道接收到地消息进入应用程序     |
| @Output         | 注解标识输出通道，发布的消息将通过该通道离开应用程序         |
| @StreamListener | 监听队列，用于消费者的队列的消息接收                         |
| @EnableBinding  | 指信道channel和exchange绑定在一起                            |

## Spring Cloud Stream演示前置条件

- RabbitMQ环境已经配置完成。
- 新建`cloud-stream-rabbitmq-provider8801`，作为生产者进行发消息模块。
- 新建`cloud-stream-rabbitmq-consumer8802`，作为消息接收模块。
- 新建`cloud-stream-rabbitmq-consumer8803`，作为消息接收模块。

## 消息驱动之生产者

### 引入pom依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
        </dependency>
```

### 配置yml

其实没有涉及到服务注册发现，但为了完整性，还是将该服务注册进服务注册中心的配置加上。

```yml
server:
  port: 8801

spring:
  application:
    name: cloud-stream-provider
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: [hostname]
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        output: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
          binder: defaultRabbit # 设置要绑定的消息服务的具体设置

eureka:
  client: # 客户端进行Eureka注册的配置
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的时间间隔（默认是30秒）
    lease-expiration-duration-in-seconds: 5 # 如果现在超过了5秒的间隔（默认是90秒）
    instance-id: send-8801.com  # 在信息列表时显示主机名称
    prefer-ip-address: true     # 访问的路径变为IP地址
```

### 主启动类

```java
@SpringBootApplication
public class StreamMQMain8801 {
    public static void main(String[] args) {
        SpringApplication.run(StreamMQMain8801.class, args);
    }
}
```

### 定义消息的推送管道

```java
@EnableBinding(Source.class) //定义消息的推送管道
public class MessageProviderImpl implements IMessageProvider {

    /*
     * 消息发送管道
     */
    @Resource
    private MessageChannel output;

    @Override
    public String send() {
        String serial = UUID.randomUUID().toString();
        output.send(MessageBuilder.withPayload(serial).build());
        System.out.println("==> serial + " + serial);
        return null;
    }
}
```

### 定义接口

```java
@RestController
public class SendMessageController {
    @Resource
    private IMessageProvider messageProvider;

    @GetMapping(value = "/sendMessage")
    public String sendMessage() {
        return messageProvider.send();
    }

}
```

### 测试

启动7001注册中心，启动8801生产者模块，接着登录rabbitMQ的web图形化界面，我们将会看到一个新建的exchange：studyExchange，这是我们在yml中配置的。

接着访问接口，`localhost:8801/sendMessage`，控制台输出serial不断，rabbitMQ中也成功发送消息，测试成功。

## 消息驱动之消费者

### 引入pom依赖

同样引入`spring-cloud-starter-stream-rabbit`依赖。

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
        </dependency>
```

### 配置yml

```yml
server:
  port: 8802

spring:
  application:
    name: cloud-stream-consumer
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: [hostname]
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        input: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为对象json，如果是文本则设置“text/plain”
          binder: defaultRabbit # 设置要绑定的消息服务的具体设置

eureka:
  client: # 客户端进行Eureka注册的配置
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的时间间隔（默认是30秒）
    lease-expiration-duration-in-seconds: 5 # 如果现在超过了5秒的间隔（默认是90秒）
    instance-id: receive-8802.com  # 在信息列表时显示主机名称
    prefer-ip-address: true     # 访问的路径变为IP地址
```

### 主启动类

```java
@SpringBootApplication
public class StreamMQMain8802 {
    public static void main(String[] args) {
        SpringApplication.run(StreamMQMain8802.class, args);
    }
}
```

### 定义消费者接口

```java
@Component
@EnableBinding(Sink.class)
public class ReceiveMessageListenerController {
    @Value("${server.port}")
    private String serverPort;

    @StreamListener(Sink.INPUT)
    public void input(Message<String> message) {
        System.out.println("消费者1号,----->接受到的消息: " + message.getPayload() + "\t  port: " + serverPort);
    }
}
```

### 测试

在刚刚两个服务启动之后，再启动刚刚创建的8802模块。

![image-20201203210012472](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/image-20201203210012472.png)

访问：`localhost:8801/sendMessage`，8802模块控制台成功打印消息。

## 分组消费与持久化

按照上面的步骤运行下来，貌似没什么问题，其实并不是？

- 如果一个生产者对应的消费者增多，同一个消息，两个消费者同时收到，会产生重复消费的问题。
- 消息如何持久化？

为了演示这个问题，我们仿照8802模块，克隆一份8803模块，并依次启动7001注册中心，8801消息生产者，8802、8803消息消费者。

![image-20201203211109661](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/image-20201203211109661.png)

我们只需访问：`localhost:8801/sendMessage`就能够显示消息重复消费的问题，对此，我们可以通过Stream中的**分组消费来**解决，同一个组的消费者存在竞争关系，只能有一个可以进行消费。

![image-20201203211434246](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/image-20201203211434246.png)

我们通过rabbitMQ的管理页面就能看到，这两个消费者默认的组流水号是不同的，解决的办法也很简单，指定他们的流水号相同即可。

我们在8802和8803中的yml中配置：`spring.cloud.stream.bindings.input.group=A`即可实现轮询消费。

![image-20201203212102137](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E5%9B%9B%EF%BC%89SpringCloudStream%E6%B6%88%E6%81%AF%E9%A9%B1%E5%8A%A8/image-20201203212102137.png)

消息持久化是很关键的步骤，如果不具备消息持久化的功能，假设某一消费者突然宕机，生产者持续发送消息，消费者无法消费，会导致消息丢失。

加上group属性之后，就已经具备了消息持久化，演示也很简单，关闭消费者服务，生产者不断发送信息，重启消费者服务，发现启动之后，将宕机时的错过的消息消费。

## [源码下载](https://tqbx.gitee.io/javablog/#/微服务学习/SpringCloud学习笔记（二）Eureka服务注册与发现?id=源码下载)

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。