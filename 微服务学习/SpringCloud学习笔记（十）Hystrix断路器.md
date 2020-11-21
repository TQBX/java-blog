## 分布式系统面临的问题

**服务雪崩**：多个微服务之间调用的时候，假设微服务A调用微服务B和微服务C，微服务B和微服务C又调用其他的微服务，这就是所谓的“扇出”。如果扇出的链路上某个微服务的调用响应时间过长或不可用，对微服务A的调用就会占用越来越多的系统资源，进行引起系统崩溃。

对于高流量的应用来说，**单一的后端依赖可能会导致所有服务器上的所有资源都在几秒钟内饱和**。比失败更糟糕的是，这些应用程序还可能导致服务之间的延迟增加，备份队列，线程和其他系统资源紧张，导致整个系统发生更多的**级联故障**。

这些都表示需要对故障和延迟进行隔离和管理，以便单个依赖关系的失败，不能取消整个应用程序或系统。

## Hystrix概述

> 官网： https://github.com/Netflix/Hystrix/wiki
>
> 目前已经进入维护模式。

### Hystrix是啥？

Hystrix是一个用于处理分布式系统的**延迟**和**容错**的开源库，在分布式系统里，许多依赖必不可免的会调用失败，比如超时、异常等，**Hystrix能够保证在一个依赖出问题的情况下，不会导致整体服务失败，避免级联鼓掌，以提高分布式系统的弹性**。

**断路器**本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似<u>熔断保险丝</u>）**向调用方返回一个符合预期的、可处理的备选响应（Fallback），而不是常见的等待或者抛出调用方无法处理的异常**，这样就**保证了服务调用方的线程不会被长时间、不必要地占用**，从而避免了故障在分布式系统中的蔓延，乃至雪崩。

### Hystrix能干啥？

- 提供保护并控制延迟和失败。
- 停止复杂的分布式系统中的级联故障。
- 快速失败并且快速恢复。
- 回退，并在可能的情况下正常降级。
- 启用近乎实时的监控，警报和操作控制。

## Hystrix重要概念

### 服务降级fallback

当服务器压力剧增的情况下，根据实际业务情况及流量，**对一些服务和页面有策略的不处理或换种简单的方式处理**，从而释放服务器资源以保证核心交易正常运作或高效运作。

### 服务熔断break

服务熔断的作用类似于我们家用的保险丝，当某服务出现不可用或响应超时的情况时，为了防止整个系统出现雪崩，**暂时停止对该服务的调用**。

### 服务限流flowlimit

限流的目的是为了保护系统不被大量请求冲垮，通过限制请求的速度来保护系统。

## Hystrix演示-构建异常环境

我们先准备环境，准备一个正常的接口方法和一个模拟延迟三秒的接口方法。

### 创建新模块，引入依赖

```java
         <!--hystrix-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <!--eureka client-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

### 编写yml

```yml
server:
  port: 8001

spring:
  application:
    name: cloud-provider-hystrix-payment # 服务名称!

eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true。
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为true才能配合ribbon使用负载均衡
    fetchRegistry: true
    service-url:
      #      #单机版
      defaultZone: http://localhost:7001/eureka
```

### 编写主启动类

```java
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
public class PaymentHystrixMain8001 {
    public static void main(String[] args) {
        SpringApplication.run(PaymentHystrixMain8001.class, args);
    }
}
```

### 编写service层

```java
@Service
public class PaymentService {

    // 正常访问
    public String paymentInfoOk(Integer id) {
        return "线程池:  " + Thread.currentThread().getName() + "  paymentInfo_OK,id:  " + id + "\t" + "O(∩_∩)O哈哈~";
    }
	// 延迟3秒
    public String paymentInfoTimeOut(Integer id) {
        //int age = 10/0;
        try {
            TimeUnit.MILLISECONDS.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "线程池:  " + Thread.currentThread().getName() + " id:  " + id + "\t" + "O(∩_∩)O哈哈~" + "  耗时(秒): ";
    }
}
```

### 编写Controller层

```java
@RestController
@Slf4j
public class PaymentController {
    @Resource
    private PaymentService paymentService;

    @Value("${server.port}")
    private String serverPort;

    @GetMapping("/payment/hystrix/ok/{id}")
    public String paymentInfoOk(@PathVariable("id") Integer id) {
        String result = paymentService.paymentInfoOk(id);
        log.info("*****result: " + result);
        return result;
    }

    @GetMapping("/payment/hystrix/timeout/{id}")
    public String paymentInfoTimeOut(@PathVariable("id") Integer id) {
        String result = paymentService.paymentInfoTimeOut(id);
        log.info("*****result: " + result);
        return result;
    }
}
```

### 测试构建的环境

依次启动：7001Eureka服务中心，8001服务，分别访问以下两个链接：

- http://localhost:8001/payment/hystrix/ok/1
- http://localhost:8001/payment/hystrix/timeout/1

### 压力测试

当并发请求量足够大的时候，微服务会集中资源去处理响应较慢的服务，导致其他本来响应较快的服务被拖累。Tomcat默认的工作线程被打满，没有多余的线程来分解压力和处理。

如果新建消费者客户端80，8001同一层次的其他接口服务被困死，80此时调用8001，客户端的响应也会变得很慢。

## Hystrix演示-降级熔断限流

