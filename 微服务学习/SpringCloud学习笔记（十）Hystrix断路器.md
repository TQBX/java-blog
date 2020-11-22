## 本篇要点

- 简单了解分布式系统目前面临的问题。
- 简单了解Hystrix服务降级、熔断、限流的概念。
- 实战服务降级和服务熔断。
- 理解Hystrix的工作流程。
- 实战使用Hystrix图形化监控页面。

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

> [Spring Cloud Netflix# Circuit Breaker:Hystrix Clients](https://docs.spring.io/spring-cloud-netflix/docs/2.2.5.RELEASE/reference/html/#circuit-breaker-hystrix-clients)

我们先准备环境，准备一个正常的接口方法和一个模拟延迟五秒的接口方法。

### 创建新模块，引入依赖

`spring-cloud-starter-netflix-hystrix`依赖引入。

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
            TimeUnit.SECONDS.sleep(5);
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

## Hystrix演示-服务降级

### 服务端服务降级

**cloud-provider-hystrix-payment8001**：在paymentInfoTimeOut方法上使用@HystrixCommand，并定义fallbackMethod方法。

```java
@HystrixCommand(fallbackMethod = "paymentInfoTimeOutHandler", commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000")
})

public String paymentInfoTimeOut(Integer id) {
    //int age = 10/0;
    try {
        TimeUnit.SECONDS.sleep(5);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "线程池:  " + Thread.currentThread().getName() + " id:  " + id + "\t" + "O(∩_∩)O哈哈~" + "  耗时(秒): ";
}

public String paymentInfoTimeOutHandler(Integer id) {
    return "线程池:  " + Thread.currentThread().getName() + "  8001系统繁忙或者运行报错，请稍后再试,id:  " + id + "\t" + "o(╥﹏╥)o";
}
```

一旦调用服务方法失败并抛出了错误信息后，会自动调用@HystrixCommand标注好的fallbackMethod调用类中指定的指定方法。

`@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")`设置了服务自身的上线时间为3s，超过3s将会调用paymentInfoTimeOutHandler方法。

主启动类加上激活注解：

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

访问：`http://localhost:8001/payment/hystrix/timeout/1`接口进行测试，3s之后页面返回fallbackMethod设置方法的结果。既然超时状况可以进行兜底处理，那异常情况呢？

我们不妨打开`int age = 10/0;`的注释，此时会抛出一个算数异常，进行测试，一旦抛出异常，也会fallback。

### 消费端服务降级

cloud-consumer-feign-hystrix-order80：为消费者端也配置服务降级。首先配置yml，开启`feign.hystrix.enabled`。

```yml
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/

feign:
  hystrix:
    enabled: true
```

使用@HystrixCommand注解设置服务降级处理方法，和等待上限时间属性。

```java
@RestController
@Slf4j
public class OrderHystirxController {
    @Resource
    private PaymentHystrixService paymentHystrixService;

    @GetMapping("/consumer/payment/hystrix/timeout/{id}")
    @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod", commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500")
    })
    public String paymentInfoTimeOut(@PathVariable("id") Integer id) {
        //int age = 10 / 0;
        return paymentHystrixService.paymentInfTimeOut(id);
    }

    public String paymentTimeOutFallbackMethod(@PathVariable("id") Integer id) {
        return "我是消费者80,对方支付系统繁忙请10秒钟后再试或者自己运行出错请检查自己,o(╥﹏╥)o";
    }

}
```

在主启动类上加上注解：

```java
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
public class OrderHystrixMain80 {
    public static void main(String[] args) {
        SpringApplication.run(OrderHystrixMain80.class, args);
    }
}
```

这时候的情景是：

- 客户端的降级策略为：将要调用服务端接口，且希望1.5s以内就能得到回应，否则就会执行paymentTimeOutFallbackMethod方法。
- 服务端的降级策略为：服务如果超过5s中才执行fallbackMethod中的方法。

### 暴露的问题

- 每个业务方法都需要一个兜底方法，代码膨胀！
- 处理方法和业务逻辑混叠在一起，混乱！

### 定义默认的fallbackMethod

我们应当按照全局异常处理的思想，定义一套全局通用的处理降级方法，再此基础之上，再分别针对不同的业务，进行定制降级。

```java
@RestController
@Slf4j
@DefaultProperties(defaultFallback = "paymentGlobalFallbackMethod")
public class OrderHystirxController {
    @Resource
    private PaymentHystrixService paymentHystrixService;

    @HystrixCommand
    @GetMapping("/consumer/payment/hystrix/timeout/{id}")
    public String paymentInfoTimeOut(@PathVariable("id") Integer id) {
        //int age = 10 / 0;
        return paymentHystrixService.paymentInfTimeOut(id);
    }

    public String paymentTimeOutFallbackMethod(@PathVariable("id") Integer id) {
        return "我是消费者80,对方支付系统繁忙请10秒钟后再试或者自己运行出错请检查自己,o(╥﹏╥)o";
    }

    /**
     * 下面是全局fallback方法
     */
    public String paymentGlobalFallbackMethod() {
        return "Global异常处理信息，请稍后再试，/(ㄒoㄒ)/~~";
    }
}
```

- 在Controller层上加上`@DefaultProperties(defaultFallback = "paymentGlobalFallbackMethod")`。定义默认的fallback处理方法。
- 如果@HystrixCommand不指定fallbackMethod，将会调用defaultFallback 。

> The `@HystrixCommand` is provided by a Netflix contrib library called [“javanica”](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica). Spring Cloud automatically wraps Spring beans with that annotation in a proxy that is connected to the Hystrix circuit breaker. The circuit breaker calculates when to open and close the circuit and what to do in case of a failure.
>
> To configure the `@HystrixCommand` you can use the `commandProperties` attribute with a list of `@HystrixProperty` annotations.

### 解决Controller层耦合

我们首先需要知道，我们所有需要调用服务端的接口的方法，其实都定义在FeignClient中。那么，我们可以@FeignClient注解中通过fallback指定同一处理降级的服务：`PaymentFallbackService`，我们让这个service实现FeignClient，针对每个方法提供不同的降级策略。

首先确保开启：

```yml
feign:
  hystrix:
    enabled: true
```

```java
@Component
@FeignClient(value = "cloud-provider-hystrix-payment",fallback = PaymentFallbackService.class)
public interface PaymentHystrixService {
    @GetMapping("/payment/hystrix/ok/{id}")
    String paymentInfOk(@PathVariable("id") Integer id);

    @GetMapping("/payment/hystrix/timeout/{id}")
    String paymentInfTimeOut(@PathVariable("id") Integer id);
}
```

```java
@Component
public class PaymentFallbackService implements PaymentHystrixService {

    @Override
    public String paymentInfOk(Integer id) {
        return "PaymentFallbackService fall back-paymentInfo_OK ,o(╥﹏╥)o";
    }

    @Override
    public String paymentInfTimeOut(Integer id) {
        return "PaymentFallbackService fall back-paymentInfo_TimeOut ,o(╥﹏╥)o";
    }
}
```

### 如何测试

1. 先后启动7001，8001。
2. 访问：`http://localhost/consumer/payment/hystrix/ok/1`结果正常。
3. 此时将8001强制关闭，此时客户端要想调用服务端必然调用失败，这时客户端的服务降级就会效果显现。

### 哪些情况会发出降级

1. 程序运行异常
2. 超时
3. 服务熔断触发服务降级
4. 线程池/信号量打满也会导致服务降级

## Hystrix演示-服务熔断

### 熔断机制概述

**熔断机制是应对雪崩效应的一种微服务链路保护机制**，当扇出链路的某个微服务出错或不可用或响应时间太长时，会进行服务的降级，进而熔断该节点微服务的调用，快速返回错误的响应信息。**当检测到该节点微服务调用响应正常后，恢复调用链路**。

SpringCloud中可以通过**Hystrix**实现熔断，Hystrix会监控微服务之间调用的状况，当失败的调用到一定阈值，缺省是5秒内20次调用失败，就会启动熔断机制。

> https://martinfowler.com/bliki/CircuitBreaker.html

### 熔断演示，配置参数

在Service层配置熔断策略：

```java
    //=====服务熔断 10s之内 10次请求有6次失败 就会开启断路器
    @HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback", commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),// 是否开启断路器
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),// 请求次数
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"), // 时间窗口期
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),// 失败率达到多少后跳闸
    })
    public String paymentCircuitBreaker(Integer id) {
        if (id < 0) {
            throw new RuntimeException("******id 不能负数");
        }
        String serialNumber = IdUtil.simpleUUID();

        return Thread.currentThread().getName() + "\t" + "调用成功，流水号: " + serialNumber;
    }
```

> @HystrixProperty中的配置项定义在：`com.netflix.hystrix.HystrixCommandProperties`
>
> 更加详细的配置介绍：https://github.com/Netflix/Hystrix/wiki/Configuration

在Controller层调用：

```java
    //====服务熔断
    @GetMapping("/payment/circuit/{id}")
    public String paymentCircuitBreaker(@PathVariable("id") Integer id) {
        String result = paymentService.paymentCircuitBreaker(id);
        log.info("****result: " + result);
        return result;
    }
```

### 测试

![image-20201122192917543](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/image-20201122192917543.png)

### 熔断类型

![img](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/state.png)

1. 熔断打开Open：请求不再进行调用当前服务，内部设置时钟一般为MTTR【平均故障处理时间】，当打开时长达到所设始终则进入半熔断状态。
2. 熔断关闭Closed：熔断关闭不会对服务进行熔断。
3. 熔断半开Half Open：部分请求根据规则调用当前服务，如果请求成功且符合规则则认为当前服务恢复正常，关闭熔断。

![circuit-breaker-1280](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/circuit-breaker-1280.png)

### 熔断啥时候起作用

涉及到断路器的三个重要参数：**快照时间窗**、**请求总数阀值**、**错误百分比阀值**。

1. 快照时间窗sleepWindowInMilliseconds：断路器确定是否打开需要统计一些请求和错误数据，而**统计的时间范围**就是快照时间窗，默认为最近的10秒。
2. 请求总数阀值requestVolumeThreshold：在快照时间窗内，**必须满足请求总数阀值才有资格熔断**。默认为20，意味着在10秒内如果该hystrix命令的调用次数不足20次，即使所有的请求都超时或其它原因失败，断路器都不会打开。
3. 错误百分比阀值errorThresholdPercentage：当请求总数在快照时间窗内超过了阀值，比如发生了30次调用，如果在这30次调用中，有超过15次发生了超时异常，也就是超过50%的错误百分比（默认错误阀值50%），断路器会打开。

### 断路器开启或关闭的条件

1. 当满足一定阈值时（默认10秒内超过20个请求次数）
2. 当失败率达到一定时（默认10秒内超过50%的请求失败）
3. 到达以上阈值，断路器将开启
4. 当开启时，所有请求将不会转发
5. 一段时间后（默认是5秒），这个时候断路器是半开状态，会让其中一个进行转发。如果成功，断路器会关闭，若失败，继续开启。重复4~5

断路器打开之后，再有请求调用的时候，将不会再调用主逻辑，而是**直接调用降级fallback**，通过断路器，**实现了自动地发现错误并将降级逻辑切换为主逻辑，减少响应延迟的效果**。

**原来的逻辑如何恢复？**

当断路器打开，对主逻辑进行熔断之后，hystrix会启动一个休眠时间窗，在这个时间窗内，降级逻辑临时的成为主逻辑，当休眠时间窗到期，断路器将进入半开状态，释放一次请求到原来的主逻辑上，如果此次请求正常返回，那么断路器将闭合，主逻辑恢复，如果这次请求依然有问题，断路器继续进入打开状态，休眠时间窗重新计时。

## Hystrix工作流程总结

<img src="img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/hystrix-command-flow-chart.png" alt="hystrix-command-flow-chart"  />

1. 构建 HystrixCommand【依赖的服务返回单个的结果】 or HystrixObservableCommand【依赖的服务返回多个操作结果】对象【1】。

2. 执行命令，以下四种方法中的一种【前两种方法仅适用于简单的HystrixCommand对象，而不适用于HystrixObservableCommand】【2】：

   1. execute()，同步，从依赖的服务返回一个单一的响应【或在发生错误的时候抛出异常】。
   2. queue()，返回Future对象，包含了服务执行结束时要返回的单一结果对象。
   3. observe()：返回Observable对象。
   4. toObservable()：返回Observable对象，也代表了操作的多个结果。

   ```java
   K             value   = command.execute();
   Future<K>     fValue  = command.queue();
   Observable<K> ohValue = command.observe();         //hot observable
   Observable<K> ocValue = command.toObservable();    //cold observable
   ```

3. 检查缓存【3】，若当前命令的请求缓存功能是被启用的，并且该命令缓存命中，那么这个缓存的响应将立即以Observable形式返回。
4. 检查断路器是否是打开状态【4】，如果断路器打开，则Hystrix不会执行命令，而是处理fallback逻辑【8】，否则检查是否有可用资源来执行命令【5】。
5. 线程池/请求队列/信号量是否已满【5】，如果命令依赖服务的专有线程池和请求队列，或信号量已经被占满，那么Hystrix也不会执行命令，而是转到第【8】步。
6. Hystrix会根据我们编写的方法来决定采取什么样的方式去请求依赖服务【6】。
   1. HystrixCommand.run()：返回一个单一的结果，或者抛出异常。
   2. HystrixObservableCommand.constract()：返回一个Observable对象来发送多个结果或通过onError发送错误通知。
7. Hystrix向断路器报告成功、失败、拒绝和超时等信息，断路器通过维护一组计数器来统计这些数据，通过这些数据来决定是否要打开断路器【7】。
8. 当命令执行失败时，Hystrix会进入fallback尝试回退处理，我们通常称为：服务降级。能够引起服务降级处理的情况：
   1. 熔断器打开。【4】
   2. 当前命令的线程池、请求队列、信号量被占满的时候。【5】
   3. HystrixCommand.run()或HystrixObservableCommand.constract()抛出异常时。【6】
9. 当Hystrix命令执行成功之后，它会将处理结果直接返回或是以Observable的形式返回。

## Hystrix图形化DashBoard监控实战

### 引入依赖

```java
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

### 设置监听9001端口

```yml
server:
  port: 9001
```

### 添加注解@EnableHystrixDashboard

```java
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashboardMain9001 {
    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardMain9001.class, args);
    }
}
```

### 测试

访问：`http://localhost:9001/hystrix`，出现以下界面说明配置成功：

![image-20201122213208322](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/image-20201122213208322.png)

### 实战监控8001服务熔断

保证8001的依赖中存在`spring-boot-starter-actuator`。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

特殊配置

```java
    /**
     * 此配置是为了服务监控而配置，与服务容错本身无关，spring cloud升级后的坑
     * ServletRegistrationBean因为springboot的默认路径不是"/hystrix.stream"，
     * 只要在自己的项目里配置上下面的servlet就可以了
     */
    @Bean
    public ServletRegistrationBean<HystrixMetricsStreamServlet> getServlet() {
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean<HystrixMetricsStreamServlet> registrationBean = new ServletRegistrationBean<>(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/hystrix.stream"); //访问路径
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
```

这时，在Hystrix界面监控框输入监控的url：`http://localhost:8001/hystrix.stream`，可能会出现错误：

```java
Origin parameter: http://localhost:8001/hystrix.stream is not in the allowed list of proxy host names.  If it should be allowed add it to hystrix.dashboard.proxyStreamAllowList.
```

提醒我们设置`hystrix.dashboard.proxyStreamAllowList`，在yml中设置呗：

```yml
server:
  port: 9001

hystrix:
  dashboard:
    proxy-stream-allow-list:
      - "localhost"
```

启动8001，按照我们测试服务熔断的流程，监控正常运行的效果：

![image-20201122215657639](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/image-20201122215657639.png)

![image-20201122220732686](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%EF%BC%89Hystrix%E6%96%AD%E8%B7%AF%E5%99%A8/image-20201122220732686.png)

- 请求示意图的圆圈越大，表示请求量越多。
- 曲线表示过去2分钟的请求变化率。
- Host、Cluster表示每秒的并发请求数。
- Hosts表示节点个数，只有一个示例，则为1。
- ThreadPools表示监控Hystrix的线程状态。

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。