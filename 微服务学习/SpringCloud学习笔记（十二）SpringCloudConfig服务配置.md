## 本篇要点

- 介绍Spring Cloud Config是什么，能够解决分布式系统中的什么问题。
- 介绍Spring Cloud Config服务端配合Git配置，以及客户端的配置。
- 介绍客户端手动动态刷新的方法。

## 分布式系统面临的问题

微服务意味着要将单体应用中的业务拆分成一个个子服务, 每个服务的粒度相对较小，因此系统中会出现大量的服务。**由于每个服务都需要必要的配置信息才能运行，所以一套集中式的、动态的配置管理设施是必不可少的。**

SpringCloud提供了`ConfigServer`来解决这个问题，我们每一个微服务自己带着一个`application.yml`, 上百个配置文件的管理

## Spring Cloud Config是什么

> https://spring.io/projects/spring-cloud-config

**SpringCloud Config为微服务架构中的微服务提供集中化的外部配置支持，配置服务器为各个不同微服务应用的所有环境提供了一个中心化的的外部配置**

![image-20201126221913704](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E5%8D%81%E4%BA%8C%EF%BC%89SpringCloud%20Config%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE/image-20201126221913704.png)

- SpringCloud Config分为**服务端和客户端**两部分。
- **服务端也称为分布式配置中心**，它是一个独立的微服务应用， 用来连接配置服务器并为客户端提供获取配置信息,加密/解密信息等访问接口
- **客户端则是通过指定的配置中心来管理应用资源**，以及与业务相关的配置内容,并在启动的时候从配置中心获取和加载配置信息配置服务器默认采用git来存储配置信息，这样就有助于对环境配置进行版本管理,并且可以通过git客户端工具来方便的管理和访问配置内容

## SpringCloud Config的作用

- 集中管理配置文件
- 不同环境不同配置，动态化的配置更新，分环境部署比如`dev/test/prod/beta/release`
- **运行期间动态调整配置，不再需要在每个服务部署的机器上编写配置文件，服务会向配置中心统一拉取配置自己的信息当配置发生变动时，服务不需要重启即可感知到配置的变化并应用新的配置**
- 将配置信息以**REST接口**的形式暴露

## SpringCloud Config整合Git搭建配置总控中心

### 在Gitee上新建仓库，并上传文件

> https://gitee.com/tqbx/spring-cloud-config

上传文件：config-dev.yml

```yml
config:
  info: "master branch,springcloud-config/config-dev.yml version=7" 
```

### 新建工程，引入依赖

新建：`cloud-config-center3344`，引入`spring-cloud-config-server`依赖。

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
```

### 配置yml

```yml
server:
  port: 3344

spring:
  application:
    name:  cloud-config-center #注册进Eureka服务器的微服务名
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/tqbx/spring-cloud-config.git #Gitee上面的git仓库名字
          ####搜索目录
          search-paths:
            - SpringCloud-Config
          force-pull: true
          username: [your username]
          password: [your password]
      ####读取分支
      label: master
#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

### 报错！Auth fail

如果你按照视频中的配置出现了Auth fail的异常，你可以尝试以下操作解决：

1. uri不使用SSL【需要认证】，而是用HTTPS。
2. 配置force-pull=true。
3. 配置username和password，可以通过以下命令查看：

```shell
$ git config user.name
$ git config user.email
```

### 主启动类加上注解

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigCenterMain3344 {
    public static void main(String[] args) {
        SpringApplication.run(ConfigCenterMain3344.class, args);
    }
}
```

### 测试访问

访问：`http://localhost:3344/master/config-dev.yml`，成功获取配置数据。 

## 配置读取规则

```yml
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml # 推荐使用 
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

## Spring Cloud Config客户端配置与测试

### 新建工程，引入依赖

新建：`cloud-config-client3355`模块，引入依赖`spring-cloud-starter-config`：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>	
```

### 使用bootstrap.yml作为配置文件

#### bootstrap.yml与application.yml的区别

- `application.yml`是用户级别的资源配置项。
- `bootstrap.yml`是系统级别的资源配置，优先级更高。

#### boostrap.yml机制

Spring Cloud会创建一个`BootstrapContext`，作为Spring应用的`ApplicationContext`的父上下文。初始化的时候，BootstrapContext负责从外部源加载配置属性并解析配置。这两个上下文共享一个从外部获取的Environment。

Bootstrap属性有高优先级，默认情况下，他们不会被本地配置覆盖。BootstrapContext和ApplicationContext有着不同的约定，所以新增了`bootstrap.yml`文件，保证他们的配置分离。

#### 如何使用？

```yml
server:
  port: 3355

spring:
  application:
    name: config-client
  cloud:
    #Config客户端配置
    config:
      label: master #分支名称
      name: config #配置文件名称
      profile: dev #读取后缀名称   
      uri: http://localhost:3344 #配置中心地址k

#服务注册到eureka地址
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

config：master分支上config-dev.yml的配置文件被读取`http://config-3344.com:3344/master/config-dev.yml`。

### 创建主启动类

```java
@EnableEurekaClient
@SpringBootApplication
public class ConfigClientMain3355 {
    public static void main(String[] args) {
        SpringApplication.run(ConfigClientMain3355.class, args);
    }
}
```

### 创建Restful接口

```java
@RestController
public class ConfigClientController {
    @Value("${config.info}")
    private String configInfo;

    @GetMapping("/configInfo")
    public String getConfigInfo() {
        return configInfo;
    }
}
```

### 启动测试

依次启动7001Eureka注册中心，3344服务端配置中心，和3355客户端配置中心。

依次访问：

- `http://localhost:7001/`
- `http://localhost:3344/master/config-dev.yml`
- `http://localhost:3355/configInfo`

客户端3355成功实现访问服务端3344从Gitee上获取配置config-info信息。

## 客户端手动动态刷新

之前的测试都已成功，但是存在一个问题：当Gitee上的配置文件发生改变，服务端3344确实能够及时随之改变，但是客户端却无法动态更新配置，除非重启！！！

那如何实现客户端动态刷新呢？以下配置修改3355模块，配置**手动刷新**的方式。

### 引入Actuator依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

### 配置yml，暴露监控端点

```yml
# 暴露监控端点
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### 在业务类上加上@RefreshScope

```java
@RestController
@RefreshScope
public class ConfigClientController {
    @Value("${config.info}")
    private String configInfo;

    @GetMapping("/configInfo")
    public String getConfigInfo() {
        return configInfo;
    }
}
```

@RefreshScope注解让其能够获悉外部配置的改变，但还需要下一步。

### 发送POST请求

```shell
$ curl -X POST "http://localhost:3355/actuator/refresh"

["config.client.version","config.info"]
```

此时，手动刷新配置的方式已经配置完成，且测试成功。

手动刷新虽然已经能够实现我们的需求，但如果服务众多，每个服务都需要执行一次post请求，未免过于麻烦，那是否有一种**一次刷新，处处通知**的策略呢？有的，可以利用消息总线，下一篇学习消息总线Bus。

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。