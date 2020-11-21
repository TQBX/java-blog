# OpenFeign服务接口调用

> 官网： https://docs.spring.io/spring-cloud-openfeign/docs/2.2.5.RELEASE/reference/html/
>
> Github：https://github.com/spring-cloud/spring-cloud-openfeign

## 本篇要点

- 简单介绍Feign与OpenFeign。
- 介绍OpenFeign使用流程。
- OpenFeign的超时演示及设置。
- OpenFeign日志打印功能演示。

## OpenFeign简介

> Feign is a declarative web service client. It makes writing web service clients easier.  To use Feign create an interface and annotate it.It has pluggable annotation support including Feign annotations and JAX-RS annotations. Feign also supports pluggable encoders and decoders. Spring Cloud adds support for Spring MVC annotations and for using the same HttpMessageConverters used by default in Spring Web. Spring Cloud integrates Ribbon and Eureka, as well as Spring Cloud LoadBalancer to provide a load-balanced http client when using Feign. 

- Feigh是一个声明式web service客户端，让编写web service客户端变得简单。
- 如何使用Feign：创建一个服务接口，添加适当注解：包括Feign注解和JAX-RS注解。
- Feign还支持可插拔的encoders和decoders。
- SpringCloud对Feign进行了封装，让其能够支持SpringMVC注解和HttpMessageConverters。
- Spring Cloud集成了Ribbon和Eureka以及Spring Cloud LoadBalancer，以在使用Feign时提供负载平衡的http客户端。

### Feign能干什么

Feign旨在使编写Java Http客户端变得更容易。

前面在使用Ribbon + RestTemplate时，利用RestTemplate对http请求的封装处理， 形成了一套模版化的调用方法。但是在实际开发中，由于对服务依赖的调用可能不止一处，往往一个接口会被多 处调用，所以通常都会针对每个微服务自行封装-些客户端类来包装这些依赖服务的调用。所以,Feign在此基础上做了进一步封装， 由他来帮助我们定义和实现依赖服务接口的定义。**在Feign的实现下我们只需创建一个接口并使用注解的方式来配置它(以前是Dao接口 上面标注Mapper注解现在是一个微服务接口 上面标注一个Feign注解即可)，即可完成对服务提供方的接口绑定，**简化了使用Spring cloud Ribbon时，自动封装服务调用客户端的开发量。

### Feign集成了Ribbon

利用Ribbon维护了Payment的服务列表信息，并且通过轮询实现了客户端的负载均衡。而与Ribbon不同的是，通过feign只需要定义服务绑定接口且以声明式的方法，优雅而简单的实现了服务调用

### Feign与OpenFeign的区别

![image-20201121162157016](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%B9%9D%EF%BC%89OpenFeign%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8/image-20201121162157016.png)

## OpenFeign使用步骤

前提：准备好Eureka服务注册中心，以及两个provider-payment服务8001和8002，我们接着使用OpenFeign代替Ribbon+RestTemplate实现的负载均衡与服务调用。

### 新建消费端模块

新建`cloud-consumer-feign-order80`模块，主要引入以下两个依赖，其他的依赖可以翻到底部源码下载查看：

```xml
        <!--openfeign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
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
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
```

### 编写主启动类

`@EnableFeignClients`激活Feign，启用客户端。

```java
/**
 * @EnableFeignClients 激活开启
 * @author summerday
 */
@SpringBootApplication
@EnableFeignClients
public class OrderFeign80Application {
    public static void main(String[] args) {
        SpringApplication.run(OrderFeign80Application.class, args);
    }
}
```

### 编写业务类

创建Feign客户端：

```java
@Component
@FeignClient(value = "cloud-payment-service")
public interface PaymentFeignClient {

    @GetMapping("/payment/{id}")
    AjaxResult getById(@PathVariable("id") Long id);
}
```

@FeignClient注解中的value值用于创建Ribbon负载均衡器或SpringCloud负载均衡器，你还可以使用url属性来指定url

上面的`load-balancer client`将发现`"cloud-payment-service"`服务的物理地址。 如果你的应用程序是Eureka客户端，则它将在Eureka服务注册表中解析该服务。 如果你不想使用Eureka，那么只需使用SimpleDiscoveryClient在外部配置中配置服务器列表即可。

### 编写消费者端的控制器

注入FeignClient实例，并调用其接口，进而调用生产者端的接口。

```java
@RestController
@Slf4j
public class PaymentFeignController {

    @Resource
    private PaymentFeignClient paymentFeignClient;

    @GetMapping("/consumer/payment/{id}")
    public AjaxResult getById(@PathVariable("id") Long id){
        return paymentFeignClient.getById(id);
    }

}
```

我们来理一理大致的流程，当我们发送`localhost/consumer/payment/1`请求的时候：

1. 先来到PaymentFeignController，执行getById方法，调用paymentFeignClient。
2. paymentFeignClient会通过我们配置的服务名【cloud-payment-service】找到对应的接口，并调用。
3. 接口回调，返回结果。

## OpenFeign超时设置

OpenFeign默认调用的服务超时时间为1s，如果超过会报错。

### 超时演示

演示代码比较简单，直接提供一个业务逻辑不存在问题，只是存在延迟问题的接口：

```java
    @GetMapping("/payment/timeOut")
    public String paymentTimeOut(){

        try {
            // 延迟3s
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return serverPort;
    }
```

客户端的代码如下：

```java
@Component
@FeignClient(value = "cloud-payment-service")
public interface PaymentFeignClient {

    @GetMapping("/payment/timeOut")
    String paymentTimeOut();
}

@RestController
@Slf4j
public class PaymentFeignController {

    @Resource
    private PaymentFeignClient paymentFeignClient;

    @GetMapping("/consumer/payment/timeOut")
    public String paymentTimeOut(){
        //openFeign ribbon客户端默认等待1s
        return paymentFeignClient.paymentTimeOut();
    }
}
```

启动Eureka服务中心，注册生产者服务端，启动消费端，访问：`localhost/consumer/payment/timeOut`，默认超过1s，将会直接在页面上抛出异常。

![image-20201121193836053](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%B9%9D%EF%BC%89OpenFeign%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8/image-20201121193836053.png)

### 超时设置

超过1s直接报错确实挺狠，但有时候真的需要消费者等待一定的时间，这怎么办呢？在yml中配置即可。

```yml
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/

#设置feign客户端超时时间(OpenFeign默认支持ribbon)
ribbon:
  #指的是建立连接后从服务器读取到可用资源所用的时间
  ReadTimeout: 5000
  #指的是建立连接所用的时间，适用于网络状况正常的情况下,两端连接所用的时间
  ConnectTimeout: 5000
```

## OpenFeign日志打印增强

Feign提供了日志打印功能，我们可以通过配置来调整日志级别，从而了解Feign中Http请求的细节。

说白了就是对**Feign接口的调用情况进行监控和输出**

### OpenFeign的日志级别

- NONE：默认的，不显示任何日志;

- BASIC：仅记录请求方法、URL、 响应状态码及执行时间;
- HEADERS：除了BASIC 中定义的信息之外，还有请求和响应的头信息;
- FULL：除了HEADERS中定义的信息之外,还有请求和响应的正文及元数据。

### 如何启动日志打印功能

```java
@Configuration
public class FeignConfig {

    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

```yml
logging:
  level:
    com.hyh.springcloud.service.PaymentFeignClient: debug
```

结果如下：

![image-20201121195417133](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%B9%9D%EF%BC%89OpenFeign%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8/image-20201121195417133.png)

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：https://gitee.com/tqbx/spring-cloud-learning，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。