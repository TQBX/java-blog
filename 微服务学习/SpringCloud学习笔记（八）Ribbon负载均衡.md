## 本篇要点

- 介绍Ribbon的基本功能。
- 介绍负载均衡的相关概念。
- 演示Ribbon负载均衡。
- 学习Ribbon默认自带的负载均衡规则。
- 学习轮询算法原理。

## Ribbon是什么？

Ribbon是Netflix发布的开源项目，主要功能是**提供客户端的软件负载均衡算法和服务调用**，将Netflix的中间层服务连接在一起。

Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。简单的说，就是在配置文件中列出`Load Balancer`（简称LB）后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随机连接等）去连接这些机器。我们很容易使用Ribbon实现自定义的负载均衡算法。

目前Ribbon项目的状态处于：维护中。

> https://github.com/Netflix/ribbon/wiki

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套**客户端负载均衡**的工具。

## LoadBalance负载均衡

负载均衡简单的说就是将用户的请求平摊的分配到多个服务上，从而达到系统的HA【高可用】。

常见的负载均衡有软件Nginx，LVS，硬件 F5等。

### Ribbon与Nginx负载均衡的区别

Nginx是服务器负载均衡，客户端所有请求都会交给nginx，nginx实现转发请求，负载均衡由服务端实现。

Ribbon是本地负载均衡，在调用微服务接口时候，会在注册中心上获取注册信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用。

### 集中式LB与进程内LB

**集中式负载均衡**：即在服务的消费方和提供方之间使用独立的LB设施(可以是硬件，如F5, 也可以是软件，如**nginx**), 由该设施负责把访问请求通过某种策略转发至服务的提供方；

**进程内负载均衡**：将LB逻辑集成到消费方，消费方从服务注册中心获知有哪些地址可用，然后自己再从这些地址中选择出一个合适的服务器。
**Ribbon就属于进程内LB**，它只是一个类库，**集成于消费方进程**，消费方通过它来获取到服务提供方的地址。

## Ribbon负载均衡演示

Ribbon是一个软负载均衡客户端组件，它可以和其他所需请求的客户端结合使用，和Eureka结合只是其中一个实例。

![image-20201121120147615](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%85%AB%EF%BC%89Ribbon%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/image-20201121120147615.png)

### Ribbon工作步骤

1. 选择EurekaServer，优先选择在同一区域内负载较少的server。
2. 根据用户指定的策略，从server获取到的服务注册列表中选择一个地址。策略包括：轮询，随机，根据响应时间加权。

### 整合Ribbon

我们要整合Ribbon，当然需要引入Ribbon响应的依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
```

事实上，我们之前Eureka的例子，通过80端口，轮询访问8001和8002端口，就是客户端负载均衡的体现，我们之前引入的依赖如下：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

其实就已经整合了Ribbon，：

```xml
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
      <version>2.2.6.RELEASE</version>
      <scope>compile</scope>
    </dependency>
```

这就是为什么我们不用显式地去引入Ribbon的依赖，我们也可以知道Ribbon的实现其实就是：**负载均衡+RestTemplate调用**。

### RestTemplate

> https://docs.spring.io/spring-framework/docs/5.2.6.RELEASE/javadoc-api/org/springframework/web/client/RestTemplate.html

getForEntity：返回对象为ResponseEntity对象，包含了响应中的一些重要信息，如响应头，响应状态码，响应体等。

getForObject：返回对象为响应体中数据转化成的对象，基本可以理解为json。

## Ribbon默认自带的负载规则IRule

`com.netflix.loadbalancer.IRule`接口定义了负载均衡的策略，包括轮询，响应时间加权等等。

```java
/**
 * Interface that defines a "Rule" for a LoadBalancer. A Rule can be thought of
 * as a Strategy for loadbalacing. Well known loadbalancing strategies include
 * Round Robin, Response Time based etc.
 * 
 * @author stonse
 * 
 */
public interface IRule{
    /*
     * choose one alive server from lb.allServers or
     * lb.upServers according to key
     * 
     * @return choosen Server object. NULL is returned if none
     *  server is available 
     */

    public Server choose(Object key);
    
    public void setLoadBalancer(ILoadBalancer lb);
    
    public ILoadBalancer getLoadBalancer();    
}
```

![image-20201121125932393](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%85%AB%EF%BC%89Ribbon%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/image-20201121125932393.png)

- `com.netflix.loadbalancer.RoundRobinRule`：轮询，最有名也是最主要的负载均衡策略。
- `com.netflix.loadbalancer.RandomRule`：随机，从存在的servers中随机找一个。
- `com.netflix.loadbalancer.RetryRule`：先按照轮询的策略获取服务，获取失败则在指定时间内获取服务。
- `com.netflix.loadbalancer.WeightedResponseTimeRule`：对RoundRobinRule的扩展，响应速度越快的权重越大，越容易被选择。
- `com.netflix.loadbalancer.BestAvailableRule`：先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务。
- `com.netflix.loadbalancer.AvailabilityFilteringRule`：先过滤故障实例，再选择并发较小的实例。
- `com.netflix.loadbalancer.ZoneAvoidanceRule`：复合判断server所在区域的性能和server的可用性选择服务器。

## Ribbon如何更改负载规则

我们需要在**@ComponentScan**扫描不到的包下定义配置类，否则该配置类就会被所有Ribbon客户端所共享，因而达不到定制的效果。

### 定制规则

包结构如下：

![image-20201121133808628](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%85%AB%EF%BC%89Ribbon%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/image-20201121133808628.png)

启动类扫描`com.hyh.springcloud`包及其子包，我们配置在`com.hyh.rules`包下。

### 标识客户端

```java
@EnableEurekaClient
@SpringBootApplication
@RibbonClient(name = "CLOUD-PAYMENT-SERVICE",configuration = MyRule.class)
public class Order80Application {

    public static void main(String[] args) {
        SpringApplication.run(Order80Application.class, args);
    }
}
```

指定为RibbonClient，name访问`CLOUD-PAYMENT-SERVICE`提供的服务，configuration指定定义的规则。

再次测试，发现负载均衡的规则已经成为随机获取server。

## Ribbon负载均衡算法

### 轮询原理

看看最重要也是最基础的轮询算法吧，大致思想就是：`rest接口第几次请求数 % 服务器集群总数量 = 实际调用服务器位置下标`，每次重启服务器后rest接口计数从1开始。

我们不妨打开源码看一下，可能会更加清楚一些：

```java
public class RoundRobinRule extends AbstractLoadBalancerRule {

    private AtomicInteger nextServerCyclicCounter;
    private static final boolean AVAILABLE_ONLY_SERVERS = true;
    private static final boolean ALL_SERVERS = false;

    private static Logger log = LoggerFactory.getLogger(RoundRobinRule.class);

    public RoundRobinRule() {
        // 初始化AtmoicInteger = 0
        nextServerCyclicCounter = new AtomicInteger(0);
    }

    public RoundRobinRule(ILoadBalancer lb) {
        this();
        // 初始化设置LoadBalancer
        setLoadBalancer(lb);
    }

    public Server choose(ILoadBalancer lb, Object key) {
        // 不存在LoadBalancer
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }
		// server代表最终会被选择的
        Server server = null;
        // 尝试的次数
        int count = 0;
        while (server == null && count++ < 10) {
            // 获取up and reachable的servers
            List<Server> reachableServers = lb.getReachableServers();
            // 所有的servers
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();
			// 不满足选择的条件，直接报错+返回
            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }
			// 原子操作，获取索引
            int nextServerIndex = incrementAndGetModulo(serverCount);
            // 取出server
            server = allServers.get(nextServerIndex);

            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }
			// 满足条件，返回server
            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }

    /**
     * Inspired by the implementation of {@link AtomicInteger#incrementAndGet()}.
     *
     * @param modulo The modulo to bound the value of the counter.
     * @return The next value.
     */
    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            // 取余
            int next = (current + 1) % modulo;
            // CAS 操作
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }

    @Override
    public Server choose(Object key) {
        return choose(getLoadBalancer(), key);
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
    }
}
```

### 尝试模拟轮询算法

```java
@Component
public class MyLoadBalancer implements LoadBalancer {

    private AtomicInteger atomicInteger = new AtomicInteger(0);

    public final int getAndIncrement(){
        int current;
        int next;
        do{
            current = this.atomicInteger.get();
            next = current >= Integer.MAX_VALUE ? 0 : current + 1;
        }while (!this.atomicInteger.compareAndSet(current,next));
        System.out.println("访问次数 next : " + next);
        return next;
    }

    @Override
    public ServiceInstance instances(List<ServiceInstance> serviceInstances) {
        int index = getAndIncrement() % serviceInstances.size();
        return serviceInstances.get(index);
    }
}

@RestController
@Slf4j
public class OrderController {
    @Resource
    private RestTemplate restTemplate;

    @Resource
    private LoadBalancer loadBalancer;

    @Resource
    private DiscoveryClient discoveryClient;
    @GetMapping("/consumer/payment/lb")
    public String getPaymentLb() {
        List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
        if (instances == null || instances.size() == 0) {
            return null;
        }
        ServiceInstance instance = loadBalancer.instances(instances);
        URI uri = instance.getUri();
        return restTemplate.getForObject(uri + "/payment/lb", String.class);

    }
}
```

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。