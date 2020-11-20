# Zookeeper的服务注册与发现

## 安装Zookeeper环境

>  Zookeeper的3.4.9

Windows和Linux环境皆可，这里介绍一下使用Docker启动Zookeeper。

直接看这篇文章就好了：[Docker安装Zookeeper以及Zk常用命令](https://www.cnblogs.com/summerday152/p/14012622.html)

## 创建Zk服务提供者模块

### 引入依赖

```xml
<!--SpringBoot整合zookeeper客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
```

### 配置yml

```yml
#8004表示注册到zookeeper服务器的支付服务提供者端口号
server:
  port: 8004
#服务别名----注册zookeeper到注册中心名称
spring:
  application:
    name: cloud-provider-payment
  cloud:
    zookeeper:
      connect-string: 121.199.16.31:2181
```

### 添加注解@EnableDiscoveryClient

```java
/**
 * @author summerday
 *
 * //@EnableDiscoveryClient该注解用于向使用consul或者zookeeper作为注册中心时注册服务
 */
@SpringBootApplication
@EnableDiscoveryClient
public class Payment8004Application {
    public static void main(String[] args) {
        SpringApplication.run(Payment8004Application.class, args);
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

    @GetMapping("/payment/zk")
    public String paymentZk() {
        return "spring cloud with zookeeper: " + serverPort + "\t" + UUID.randomUUID().toString();
    }
}
```

我们希望将这个模块注册进Zookeeper注册中心。

### 测试，发现jar包冲突

启动程序，我们发现程序直接报错，并且戛然而止。

![image-20201120192032957](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%94%EF%BC%89Zookeeper%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120192032957.png)

经过排查，可以发现是jar包发生了冲突，原来我们引入的依赖自带着一个`zookeeper-3.5.3-beta.jar`，而我们的zookeeper是3.4.9的。

![image-20201120192158457](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%94%EF%BC%89Zookeeper%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120192158457.png)

## 解决jar包冲突

### zookeeper版本

**方法一、卸载现有的zookeeper，安装3.5.3版本的即可**

当然，这个方法可以解决冲突，但是好好地把已安装的zk卸载了，不够妥当。

**方法二、排除jar包依赖，引入3.4.9包**

```xml
<!--SpringBoot整合zookeeper客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    <!--先排除自带的zookeeper3.5.3-->
    <exclusions>
        <exclusion>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!--添加zookeeper3.4.9版本-->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
</dependency>
```

### 日志框架多绑定

如果这样你以为就成功了，非也，因为还有存在冲突的情况，如日志冲突：

```java
SLF4J: Class path contains multiple SLF4J bindings.
```

![image-20201120195208625](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%94%EF%BC%89Zookeeper%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120195208625.png)

同理，我们排除掉`slf4j-log4j12`就行了。

```xml
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.9</version>
            <!--排除slf4j-log4j12-->
            <exclusions>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

## 继续测试

在zookeeper中执行命令：

```shell
[zk: localhost:2181(CONNECTED) 7] ls / # 查看根目录下的节点
[services, zookeeper]
[zk: localhost:2181(CONNECTED) 8] ls /services  # 查看/services目录下的节点，成功入驻
[cloud-provider-payment] # yml中配置的spring.application.name
[zk: localhost:2181(CONNECTED) 9] ls /services/cloud-provider-payment
[b780fc19-2823-4420-9937-3959de87720b] # 流水号
[zk: localhost:2181(CONNECTED) 10] get /services/cloud-provider-payment/b780fc19-2823-4420-9937-3959de87720b # 获取znode结构
{"name":"cloud-provider-payment","id":"b780fc19-2823-4420-9937-3959de87720b","address":"DESKTOP-QFK0MBG","port":8004,"sslPort":null,"payload":{"@class":"org.springframework.cloud.zookeeper.discovery.ZookeeperInstance","id":"application-1","name":"cloud-provider-payment","metadata":{"instance_status":"UP"}},"registrationTimeUTC":1605873188009,"serviceType":"DYNAMIC","uriSpec":{"parts":[{"value":"scheme","variable":true},{"value":"://","variable":false},{"value":"address","variable":true},{"value":":","variable":false},{"value":"port","variable":true}]}}
cZxid = 0x31
ctime = Fri Nov 20 11:53:09 GMT 2020
mZxid = 0x31
mtime = Fri Nov 20 11:53:09 GMT 2020
pZxid = 0x31
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x175e40811430010
dataLength = 558
numChildren = 0
```

![image-20201120200230461](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%94%EF%BC%89Zookeeper%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120200230461.png)

此时，我们的服务已经成功注册进zookeeper注册中心。

## Zookeeper注册的服务是临时节点

我们直到Eureka默认是开启自我保护机制的：**宁可同时保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例**。

而Zookeeper不同，一旦你服务停止，在下一次心跳的时候，服务会被剔除，服务可以再次连接，但流水号会发生改变。

![image-20201120201419130](img/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%88%E4%BA%94%EF%BC%89Zookeeper%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/image-20201120201419130.png)

## 创建Zk服务消费者模块

pom，yml，启动类注解差不太多，这里就不赘述了，感兴趣可以查看仓库代码：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，以标签的形式详细区分每个步骤。

```java
@RestController
@Slf4j
public class OrderZKController
{
    public static final String INVOKE_URL = "http://cloud-provider-payment";

    @Resource
    private RestTemplate restTemplate;

    @LoadBalanced // 不加该注解，会导致：java.net.UnknownHostException: cloud-provider-payment
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }

    @GetMapping(value = "/consumer/payment/zk")
    public String paymentInfo()
    {
        return restTemplate.getForObject(INVOKE_URL+"/payment/zk",String.class);
    }
}
```

当我们同时启动消费者和提供者，zookeeper中就会注册进两个service。消费者访问`localhost/consumer/payment/zk`，将会调用提供者的接口，完成需求。

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。