# SpringCloud Gateway

## SpringCloud Gateway概述

### 是什么？

> Spring Cloud Gateway is an intelligent and programmable router based on Project Reactor.
>
> 官网：https://spring.io/projects/spring-cloud-gateway

![image-20201122224449176](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89Zuul%E6%9C%8D%E5%8A%A1%E7%BD%91%E5%85%B3/image-20201122224449176.png)

在云架构中运行着众多客户端和服务端，API网关的存在提供了保护和路由消息，隐藏服务，限制负载等等功能。

`Spring Cloud Gateway`提供了一个在Spring生态系统之上构建的API网关，包括：Spring 5，Spring Boot 2和Project Reactor。 Spring Cloud Gateway旨在提供一种简单而有效的方法来对API进行路由，提供精准控制。

为了提升网关的性能，Spring Cloud Gateway是基于WebFlux框架实现的，而WebFlux框架底层则使用了**高性能的Reactor模式通信框架Netty**。

SpringCloud Gateway的目标是：提供统一的路由方式且基于FilterChain的方式提供网关基本的功能，例如：安全，监控/指标，弹性，限流等等。

### 有哪些特性？

- 基于SpringFramework5，ProjectReactor和SpringBoot2.0进行构建
- 能够匹配任何任何请求属性
- 可以对路由指定Predicates和Filters
- 集成断路器
- 集成Spring Cloud服务发现
- 易于编写的Predicates和Filters
- 支持请求限流
- 支持路径重写

## 三大概念

路由：路由是构建网关的基本模块，它由ID，目标URI，一系列的断言Predicates和过滤器Filters组成，如果断言为true，则匹配该路由。

断言：参考Java8的java.util.function.Predicate，开发人员可以匹配HTTP请求中的所有内容，例如请求头或请求参数，如果请求与断言相匹配则进行路由。

过滤：Spring框架中GatewayFilter的实例，使用过滤器，可以载请求被路由前或者后对请求进行修改。

> 一个web请求，通过一些匹配条件，定位到真正的服务节点，在这个转发的过程中，进行一些精细化的控制。
>
> - predicate就是匹配条件
> - filter就是拦截器
> - predicate + filter + 目标uri实现路由route

## 工作流程

下图总体上描述了Spring Cloud Gateway的工作流程：

![Spring Cloud Gateway Diagram](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%E6%9C%8D%E5%8A%A1%E7%BD%91%E5%85%B3Gateway/spring_cloud_gateway_diagram.png)

1. 客户端向Spring Cloud Gateway发出请求。 

2. 如果Gateway Handler Mapping确定请求与路由匹配，则将其发送到Gateway Web Handler。 
3. Handler通过指定的过滤器链将请求发送到我们实际的服务执行业务逻辑，然后返回。
4. 过滤器由虚线分隔的原因是，过滤器可以在发送代理请求之前或之后执行逻辑。 

> 核心：路由转发+过滤器链

## Gateway服务搭建

### 创建模块，引入依赖

引入`spring-cloud-starter-gateway`核心组件，其次，这个服务本身也许要注册进Eureka，因此引入`spring-cloud-starter-netflix-eureka-client`。

```xml
        <!--gateway-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <!--eureka-client-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

> 注意：不能引入spring-boot-starter-web依赖

### 编写yml

```yml
server:
  port: 9527

spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true #开启从注册中心动态创建路由的功能，利用微服务名进行路由
      routes:
        - id: payment_routh #payment_route    #路由的ID，没有固定规则但要求唯一，建议配合服务名
          #uri: http://localhost:8001          #匹配后提供服务的路由地址
          uri: lb://cloud-payment-service #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/**         # 断言，路径相匹配的进行路由

        - id: payment_routh2 #payment_route    #路由的ID，没有固定规则但要求唯一，建议配合服务名
          #uri: http://localhost:8001          #匹配后提供服务的路由地址
          uri: lb://cloud-payment-service #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/lb/**         # 断言，路径相匹配的进行路由
            #- After=2020-02-21T15:51:37.485+08:00[Asia/Shanghai]
            #- Cookie=username,zzyy
            #- Header=X-Request-Id, \d+  # 请求头要有X-Request-Id属性并且值为整数的正则表达式

eureka:
  instance:
    hostname: cloud-gateway-service
    instance-id: gateway9527
    prefer-ip-address: true
  client: #服务提供者provider注册进eureka服务列表内
    service-url:
      register-with-eureka: true
      fetch-registry: true
      defaultZone: http://eureka7001.com:7001/eureka
```

这段yml配置内容比较多：

- 确定该服务的端口号9527。
- 确定了application.name为cloud-gateway。
- 开启从注册中心动态创建路由的功能，利用微服务名进行路由。
- 定义了两段路由，每段路由标明：id，uri和Predicate。
- 将该服务注册进eureka的服务列表。

### 主启动类

```java
@SpringBootApplication
@EnableEurekaClient
public class GatewayMain9527 {
    public static void main(String[] args) {
        SpringApplication.run(GatewayMain9527.class, args);
    }
}
```

### 测试启动

依次启动7001注册中心，8001服务和9527网关服务，访问`localhost:7001/`

![image-20201123003519657](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%B8%80%EF%BC%89%E6%9C%8D%E5%8A%A1%E7%BD%91%E5%85%B3Gateway/image-20201123003519657.png)

由于网关的存在，我们可以访问：`http://localhost:9527/payment/1`，能够成功访问到8001服务的接口。

## Gateway的网关配置方式

### yml配置

我们之前已经配置过了：

```yml
spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true #开启从注册中心动态创建路由的功能，利用微服务名进行路由
      routes:
        - id: payment_routh #payment_route    #路由的ID，没有固定规则但要求唯一，建议配合服务名
          #uri: http://localhost:8001          #匹配后提供服务的路由地址
          uri: lb://cloud-payment-service #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/**         # 断言，路径相匹配的进行路由
```



### 注入RouteLocator的Bean

```java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder routeLocatorBuilder) {

        RouteLocatorBuilder.Builder routes = routeLocatorBuilder.routes();

        routes.route("path_route",
                r -> r.path("/guonei")
                        .uri("http://news.baidu.com/guonei")).build();

        return routes.build();
    }
}
```

