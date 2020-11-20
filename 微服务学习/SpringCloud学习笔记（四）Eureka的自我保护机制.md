[toc]

# Eureka的自我保护机制

## 本篇要点

- 介绍Eureka的自我保护机制。
- 介绍CAP原则。
- 介绍为什么需要自我保护。
- 介绍如何禁止自我保护机制

## Eureka的自我保护

**保护模式主要用于一组客户端和Eureka Server之间存在网络分区场景下的保护**。一旦进入保护模式，Eureka Server将会尝试保护其服务注册表中的信息，不再删除服务注册表中的数据，也就是**不会注销任何微服务**。

我们之前在注册之后，强项停止某个微服务，就会出现下面这段红字，此时就代表Eureka已经进入保护模式。

![开启保护机制](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%9B%9B%EF%BC%89Eureka%E7%9A%84%E8%87%AA%E6%88%91%E4%BF%9D%E6%8A%A4%E6%9C%BA%E5%88%B6/image-20201118214928743.png)

简单来说：<u>如果某个时刻某个微服务不可用了，Eureka不会立即对其进行清理，反而会依旧对该微服务信息进行保存。</u>属于分布式CAP原则的AP分支【可用性和分区容忍性】。

## CAP是啥?

直接看看百度百科给出的解释吧：

> CAP原则又称CAP定理，指的是在一个分布式系统中，[一致性](https://baike.baidu.com/item/一致性/9840083)（Consistency）、[可用性](https://baike.baidu.com/item/可用性/109628)（Availability）、[分区容错性](https://baike.baidu.com/item/分区容错性/23734073)（Partition tolerance）。CAP 原则指的是，这三个要素最多只能同时实现两点，不可能三者兼顾。

- 一致性（C）：在**分布式系统**中的所有数据备份，**在同一时刻是否同样的值**。（等同于所有节点访问同一份最新的数据副本）

- 可用性（A）：在集群中一部分节点**故障**后，集群整体**是否还能响应**客户端的读写请求。（对数据更新具备高可用性）

- 分区容忍性（P）：以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择。

> **CAP原则的精髓就是要么AP，要么CP，要么AC，但是不存在CAP**。如果在某个分布式系统中数据无副本， 那么系统必然满足强一致性条件， 因为只有独一数据，不会出现数据不一致的情况，此时C和P两要素具备，但是如果系统发生了网络分区状况或者宕机，必然导致某些数据不可以访问，此时可用性条件就不能被满足，即在此情况下获得了CP系统，但是CAP不可同时满足。

## 为什么会产生Eureka的自我保护机制

默认情况下，如果EurekaServer在一定时间内没有收到某个微服务实例的心跳，EurekaServer将会注销该实例【默认90s】。但是当网络分区故障发生【延时、卡顿、拥挤】时候，微服务与EurekaServer之间**无法正常通信**，以上行为可能变得非常危险了。

此时微服务本身其实是健康的，本不应该注销这个微服务，Eureka通过**自我保护模式**来解决这个问题：**当EurekaServer节点在短时间内丢失过多客户端时候(可能发生了网络分区故障)**，那么这个节点就会进入自我保护模式了。

综上：自我保护模式是一种应对网络异常的安全保护措施，它的涉及哲学就是**宁可同时保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例**。老话反说就是：宁可放过三千，也不错杀一个。自我保护模式让Eureka集群更加健壮稳定。

## 如何禁止自我保护

自我保护是可配置的，默认是开启的，我们也可以通过配置禁止它，这里演示一下如何禁止。

这一项配置由`eureka.client.enable-self-preservation`决定，我们让它变为false即可。

```properties
eureka.client.server.enable-self-preservation=false # 默认是true的
```

关闭之后，我们再次启动`server7001`，访问`localhost:7001`，出现以下警示：

![image-20201120113432384](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%9B%9B%EF%BC%89Eureka%E7%9A%84%E8%87%AA%E6%88%91%E4%BF%9D%E6%8A%A4%E6%9C%BA%E5%88%B6/image-20201120113432384.png)

表明注册中心的自我保护模式已经成功关闭。

为了进行测试，我们使用单机版的Eureka环境，将`cloud-eureka-server7001`的配置改为如下：

```yml
eureka:
  instance:
    hostname: eureka7001.com #eureka服务端的实例名称
  client:
    register-with-eureka: false     #false表示不向注册中心注册自己。
    fetch-registry: false     #false表示自己端就是注册中心，我的职责就是维护服务实例，并不需要去检索服务
    service-url:
      # 设置与Eureka Server交互的地址查询服务和注册服务都需要依赖这个地址
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    #关闭自我保护机制，保证不可用服务被及时踢除
    enable-self-preservation: false
    # 清理增量队列中过期的频率
    eviction-interval-timer-in-ms: 2000
```

- 关闭自我保护机制`enable-self-preservation`。
- 每2秒清理一次失效的服务：`eviction-interval-timer-in-ms`。

为了能更快地看到效果，将`cloud-provider-payment8001` 的配置也进行修改：

```yml
eureka:
  client:
    #表示是否将自己注册进EurekaServer默认为true。
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为true才能配合ribbon使用负载均衡
    fetchRegistry: true
    service-url:
      #单机版
      defaultZone: http://localhost:7001/eureka
  instance:
    instance-id: payment8001
    #访问路径可以显示IP地址
    prefer-ip-address: true
    #Eureka客户端向服务端发送心跳的时间间隔，单位为秒(默认是30秒)
    lease-renewal-interval-in-seconds: 1
    #Eureka服务端在收到最后一次心跳后等待时间上限，单位为秒(默认是90秒)，超时将剔除服务
    lease-expiration-duration-in-seconds: 2
```

- 设置Eureka客户端向服务端发送心跳的时间间隔为1s。
- Eureka服务端在收到最后一次心跳后等待时间上限为2s。

接着我们先启动7001，才启动8001：

![image-20201120114722163](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%9B%9B%EF%BC%89Eureka%E7%9A%84%E8%87%AA%E6%88%91%E4%BF%9D%E6%8A%A4%E6%9C%BA%E5%88%B6/image-20201120114722163.png)

关闭8001：

![image-20201120114750719](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%9B%9B%EF%BC%89Eureka%E7%9A%84%E8%87%AA%E6%88%91%E4%BF%9D%E6%8A%A4%E6%9C%BA%E5%88%B6/image-20201120114750719.png)

此时，测试成功，一旦关闭自我保护模式，只要服务因为各种原因出现通信异常，服务端不会再保存他们的信息，而是选择剔除 。

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。

