## 本篇要点

- 介绍Nacos作为服务配置中心的案例。
- 介绍namespace、group、DataId三种方案的配置读取。

## Nacos服务配置中心之基础配置

### 新建模块

新建：`cloudalibaba-config-nacos-client3377`，引入依赖：

```xml
        <!--nacos-config-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <!--nacos-discovery-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
```

### 配置bootstrap.yml

Nacos和Spring Cloud config一样，在项目初始化的时候，要保证**先从配置中心进行配置拉取**，拉取配置之后，才能保证项目的正常启动。

另外，SpringBoot中配置文件的加载，**bootstrap.yml优先于application.yml**。

```yml
# nacos配置
server:
  port: 3377

spring:
  application:
    name: nacos-config-client # 构成 Nacos 配置管理 dataId字段的一部分
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #Nacos服务注册中心地址
      config:
        server-addr: localhost:8848 #Nacos作为配置中心地址
        file-extension: yaml #指定yaml格式的配置
```

配置之后，3377服务将从localhost:8848上读取后缀名为yaml的配置文件。

### 配置application.yml

```yml
spring:
  profiles:
    active: dev #表示开发环境
```

### 主启动类

```java
@EnableDiscoveryClient
@SpringBootApplication
public class NacosConfigClientMain3377 {
    public static void main(String[] args) {
        SpringApplication.run(NacosConfigClientMain3377.class, args);
    }
}
```

### 服务接口

```java
@RestController
@RefreshScope //支持Nacos的动态刷新功能。
public class ConfigClientController {

    @Value("${config.info}")
    private String configInfo;

    @GetMapping("/config/info")
    public String getConfigInfo() {
        return configInfo;
    }
}
```

### dataId的完整格式及新建配置

在 Nacos Spring Cloud 中，`dataId` 的完整格式如下：

```plain
${prefix}-${spring.profiles.active}.${file-extension}
```

- `prefix` 默认为 `spring.application.name` 的值，也可以通过配置项 `spring.cloud.nacos.config.prefix`来配置。
- `spring.profiles.active` 即为当前环境对应的 profile，详情可以参考 [Spring Boot文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html#boot-features-profiles)。 **注意：当 `spring.profiles.active` 为空时，对应的连接符 `-` 也将不存在，dataId 的拼接格式变成 `${prefix}.${file-extension}`**
- `file-exetension` 为配置内容的数据格式，可以通过配置项 `spring.cloud.nacos.config.file-extension` 来配置。目前只支持 `properties` 和 `yaml` 类型。

综上所述，按照我们的配置，最后的dataId结果应该为：

```plain
nacos-config-client-dev.yaml
```

我们选中配置列表，选择新建配置，DataID就是我们刚刚得到的`nacos-config-client-dev.yaml`。

![image-20201206005316305](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/image-20201206005316305.png)

新建配置完成之后是这样：

![image-20201206005440348](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/image-20201206005440348.png)

### 测试

运行3377服务，调用接口`http://localhost:3377/config/info`测试配置读取是否成功。

另外，它支持动态刷新，当我们修改手动修改配置中心数据时，修改的配置会被动态刷新，自动读取。

## Nacos服务配置中心之分类配置

### 解决问题

1. 实际开发中，一个系统会准备多个环境，如dev开发环境，test测试环境，prod生产环境等，如何保证指定环境启动时服务能正确读取到Nacos上相应环境的配置文件？
2. 一个大型分布式微服务系统会有很多微服务子项目，每个微服务项目都会有相应的开发环境、测试环境等，如何管理这些微服务配置呢？

### 命名空间、DataId和Group的关系

Namespace默认为空串，公共命名空间（public），分组默认是DEFAULT_GROUP。

![image-20201206010843820](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/image-20201206010843820.png)

Nacos的数据模型如下：

![nacos_data_model](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/1561217857314-95ab332c-acfb-40b2-957a-aae26c2b5d71.jpeg)

namespace用于区分部署环境【开发、测试、生产】，创建三个不同的namespace相互隔离。

Group可以把不同的微服务划分到同一个分组中。

Service可以包含多个Cluster集群，Nacos默认Cluster是DEFAULT，Cluster是对指定微服务的一个虚拟划分。

Instance是微服务的实例。

## 三种方案的加载配置

### Data Id的方案

> 保证命名空间相同，分组相同，只有Data Id不同

指定`spring.profile.active`和配置文件的`DataId`来使不同环境下读取不同的配置。为了演示这个效果，我们总共新建以下两个配置，保证它们命名空间相同，分组相同，只有Data Id不同：

```plain
nacos-config-client-dev.yaml
nacos-config-client-test.yaml
```

![image-20201206141810315](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/image-20201206141810315.png)



通过`spring.profile.active`属性就能进行多环境下配置文件的读取，刚刚已经测试过dev环境，我们测试以下test环境，是否能够读取到：`nacos-config-client-test.yaml`的配置呢，答案是肯定的，可以访问：`http://localhost:3377/config/info`测试一下。

```yml
spring:
  profles:
    active: test #表示测试环境
```

### Group方案

> 保证命名空间相同，Data Id相同，只有分组不同

![image-20201206143122184](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/image-20201206143122184.png)

注意，这里我们需要在application.yml中指定profile为info，在bootstrap.yml指定group。

```yml
## bootstrap.yml
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #Nacos服务注册中心地址
      config:
        server-addr: localhost:8848 #Nacos作为配置中心地址
        file-extension: yaml #指定yaml格式的配置
        group: DEV_GROUP
## application.yml
spring:
  profiles:
    active: info
```

测试方法不用多说，在TEST_GROUP和DEV_GROUP之间切换，再访问接口即可。

### namespace方案

> 保证命名空间不同

新建两个命名空间：dev和test。

![image-20201206144559888](img/SpringCloudAlibaba%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%EF%BC%9ANacos%E6%9C%8D%E5%8A%A1%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83/image-20201206144559888.png)

如果需要指定命名空间，则指定yml中的namespace属性即可。

```yml
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #Nacos服务注册中心地址
      config:
        server-addr: localhost:8848 #Nacos作为配置中心地址
        file-extension: yaml #指定yaml格式的配置
        group: TEST_GROUP
        namespace: 43d2f092-e338-4d31-b797-77466bdd8c8f
        
spring:
  profiles:
    active: dev #表示开发环境
```

将会从命名空间ID为`43d2f092-e338-4d31-b797-77466bdd8c8f`的`TEST_GROUP`组中，读取`nacos-config-client-dev`的配置文件。

## 源码下载

本系列文章为《尚硅谷SpringCloud教程》的学习笔记【版本稍微有些不同，后续遇到bug再做相关说明】，主要做一个长期的记录，为以后学习的同学提供示例，代码同步更新到Gitee：[https://gitee.com/tqbx/spring-cloud-learning](https://gitee.com/tqbx/spring-cloud-learning)，并且以标签的形式详细区分每个步骤，这个系列文章也会同步更新。