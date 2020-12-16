## Actuator是什么？

> 官网：[Spring Boot Actuator: Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready)

先从官网摘几句文绉绉的解释：

- SpringBoot可以帮助你再将应用程序推向生产环境时对其进行监视和管理。

- 你可以选择使用http断点或JMX来管理和监视应用程序。

- 审计【auditing】、健康【health】和指标收集【metrics gathering】可以自动应用到你的应用程序中。

总结一下就是：Actuator就是用来监控你的应用程序的。

## 快速开始

### 引入依赖

```xml
        <!--Actuator-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--Spring MVC 自动化配置-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```

一般来说这两个依赖都是相互配合的。

### yml与自动配置

下面是一段yml的配置：

```yml
server:
  port: 8081
management:
  endpoints:
    web:
      base-path: /actuator # Actuator 提供的 API 接口的根目录
      exposure:
        include: '*'  # 需要开放的端点
        exclude: # 需要排除的端点
```

- `management.endpoints.web`对应的配置类是：`org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointProperties`。
- 相关的自动配置在`org.springframework.boot.actuate.autoconfigure.endpoint.web.WebEndpointAutoConfiguration`中完成，感兴趣的同学可以查看一下源码。

- basePath是Actuator提供的API接口的根目录，默认配置就是`/actuator`。
- Exposure.include表示需要开放的端点，默认只打开`health`和`info`两个断点，设置`*`可以开放所有端点 。
- Exposure.include表示排除的端点，设置`*`为排除所有的端点。

### 主程序类

```java
@SpringBootApplication
public class SpringBootActuatorApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootActuatorApplication.class, args);
    }

}
```

### 测试

访问`http://localhost:8081/actuator/health`，得到应用的健康信息如下：

```json
{
  status: "UP"  // 开启
}
```

## Endpoints

由于我们设置开放了所有的Endpoints，启动程序时，我们能够看到控制台输出：

```plain
[  restartedMain] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 13 endpoint(s) beneath base path '/actuator'
```

说的很清楚，总共暴露了内置的13个端点，这些端点，SpringBoot文档已经交代地非常详细啦。

其实内置的端点不止13个，官方文档说：每个端点可以被`enabled or disabled`和`exposed over HTTP or JMX`。只有当端点同时enabled和exposed的时候才可用，内置的端点也只有在自动配置生效的时候才可用。

大多数的应用程序会选择HTTP暴露端点，我们可以通过`/actuator/health`访问ID为`health`的端点。

### 官方列举的所有端点列表

JMX和web共有的

| ID                 | Description                                                  |
| :----------------- | :----------------------------------------------------------- |
| `auditevents`      | 显示当前应用程序的审计事件信息，需要 `AuditEventRepository` bean. |
| `beans`            | 展示你应用程序中Spring Beans的完整列表                       |
| `caches`           | 显示可用缓存信息                                             |
| `conditions`       | 显示自动装配类的状态及是否匹配                               |
| `configprops`      | 显示所有`@ConfigurationProperties`列表                       |
| `env`              | 显示`ConfigurableEnvironment`中的属性                        |
| `flyway`           | 显示Flyway数据库迁移信息，需要一个或多个 `Flyway` beans      |
| `health`           | 显示应用的健康信息（未认证只显示status，认证显示全部信息详情） |
| `httptrace`        | 显示HTTP跟踪信息（默认显示最后100个HTTP请求 - 响应交换），需要一个 `HttpTraceRepository` bean. |
| `info`             | 显示任意的应用信息                                           |
| `integrationgraph` | 显示Spring Integration图，需要 `spring-integration-core`依赖 |
| `loggers`          | 显示和修改应用程序中日志记录器的配置。                       |
| `liquibase`        | 展示Liquibase 数据库迁移，需要一个或多个 `Liquibase` beans.  |
| `metrics`          | 展示当前应用的 metrics 信息                                  |
| `mappings`         | 显示所有`@RequestMapping` 路径集列表                         |
| `scheduledtasks`   | 显示应用程序中的计划任务                                     |
| `sessions`         | 允许从Spring会话支持的会话存储中检索和删除用户会话。 需要一个 `Servlet-based web application` using Spring Session. |
| `shutdown`         | 允许应用以优雅的方式关闭（默认情况下不启用）                 |
| `startup`          | 显示由`ApplicationStartup`收集的启动步骤. 将 `SpringApplication` 配置为`BufferingApplicationStartup`. |
| `threaddump`       | 执行一个线程dump                                             |

如果你的是web应用 (Spring MVC, Spring WebFlux, or Jersey)，你可以使用下面这些额外的端点:

| ID           | Description                                                  |
| :----------- | :----------------------------------------------------------- |
| `heapdump`   | 返回一个 `hprof` 堆转储文件                                  |
| `jolokia`    | 通过HTTP公开JMXbean 当Jolokia在类路径上时，不适用于WebFlux). 需要 `jolokia-core`依赖 |
| `logfile`    | 返回日志文件的内容(如果 `logging.file.name` 或`logging.file.path` 属性已经被设置) 支持使用HTTP`range`标头来检索部分日志文件的内容 |
| `prometheus` | 以Prometheus服务器可以抓取的格式公开指标。 需要依赖于`micrometer-registry-prometheus`。 |

### 启动端点

如果你希望启动某个端点，你可以按照`management.endpoint.<id>.enabled`的配置启动，以`shutdown`端点为例：

```yml
management:
  endpoint:
    shutdown:
      enabled: true
```

如果你只希望启动某个端点，你可以向下面这样：

```yml
management:
  endpoints:
    enabled-by-default: false # 不启用默认配置
  endpoint:
    info:
      enabled: true
```

这个示例，只会启用info端点。

### 暴露端点

端点可能会包含铭感信息，考虑暴露的时候应当慎重考虑。

- web程序默认暴露的端点有且仅有两个：`health`和`info`。

- JMX程序默认暴露所有的端点。

我们可以使用`include`和`exclude`属性来控制端点是否需要暴露。下面是两个例子

一、不让JMX暴露所有端点，只让它暴露`health`和`info`两个端点。

```yml
management:
  endpoints:
    jmx:
      exposure:
        include: "health,info"
```

二、暴露web除`env`和`beans`之外的所有端点。

```yml
management:
  endpoints:
    web:
      exposure:
        include: "*"
        exclude: "env,beans"
```

```properties
management.endpoints.web.exposure.include=*  
management.endpoints.web.exposure.exclude=env,beans
```

### 配置端点

端点将会自动缓存对不带任何参数的读取操作的响应。你可以使用`cache.time-to-live`属性配置。

以下示例将Bean端点的缓存的生存时间设置为10秒。

```yml
management:
  endpoint:
    beans:
      cache:
        time-to-live: "10s"
```

### 发现页面

默认你可以访问`/actuator`页面获得所有的端点信息：

```json
{
  _links: {
    self: {
      href: "http://localhost:8081/actuator",
      templated: false
    },
    health: {
      href: "http://localhost:8081/actuator/health",
      templated: false
    },
    health-path: {
      href: "http://localhost:8081/actuator/health/{*path}",
      templated: true
    },
    info: {
      href: "http://localhost:8081/actuator/info",
      templated: false
    }
  }
}
```

### 跨域支持

默认情况下，CORS支持是禁用的，并且仅在设置了`management.endpoints.web.cors.allowed-origins`属性后才启用。 以下配置允许来自example.com域的GET和POST调用：

```yml
management:
  endpoints:
    web:
      cors:
        allowed-origins: "https://example.com"
        allowed-methods: "GET,POST"
```

完整的选项可以查看：[CorsEndpointProperties](https://github.com/spring-projects/spring-boot/tree/v2.4.1/spring-boot-project/spring-boot-actuator-autoconfigure/src/main/java/org/springframework/boot/actuate/autoconfigure/endpoint/web/CorsEndpointProperties.java)

## 实现一个定义的端点

我们直接来看一个例子吧：

```java
@Component  // 让Spring容器管理
@Endpoint(id = "myEndPoint") //web和jmx公用， @WebEndpoint表示指定web
public class MyEndpoint {

    Runtime runtime = Runtime.getRuntime();

    @ReadOperation // 读操作
    public Map<String, Object> getData() {
        Map<String, Object> map = new HashMap<>();
        map.put("available-processors", runtime.availableProcessors());
        map.put("free-memory", runtime.freeMemory());
        map.put("max-memory", runtime.maxMemory());
        map.put("total-memory", runtime.totalMemory());
        return map;
    }
}
```

- `@Component`表明让这个bean给Spring容器管理。
- `@Endpoint`标注这是个端点，类似的还有`@WebEndpoint`标注的是特指web程序。
- `id = "myEndPoint"`限制访问端点的路径：`/actuator/myEndPoint`。
- `@ReadOperation`规定HTTP请求的方法为GET，`@ReadOperation`为POST，`@DeleteOperation`为DELETE。

接下来我们测试一下，再测试之前，不要忘了yml中启动设置：

```yml
server:
  port: 8081

management:
  endpoints:
    web:
      exposure:
        include: '*' # 需要开放的端点。默认值只打开 health 和 info 两个端点。通过设置 * 
    enabled-by-default: false
  endpoint:
    myEndPoint:
      enabled: true
```

我们暂时关闭其他所有的端点，只关注myEndPoint端点，接着启动程序，访问：`http://localhost:8081/actuator/myEndPoint`，获得json信息。

```json
{
    max-memory: 3787980800,
    free-memory: 235775968,
    available-processors: 4,
    total-memory: 298844160
}
```

实际上如果是用于SpringMVC或Spring WebFlux，我们可以使用`@RestControllerEndpoint`或`@ControllerEndpoint`注解，定义一个更符合我们平时web开发模式的端点，你可以看一下下面这个例子。

```java
@Component
@RestControllerEndpoint(id = "web")
public class MyWebEndPoint {
    @GetMapping("/")
    public Map<String, Object> getData() {
		// ...
        return map;
    }
}
```

## Health端点

端点这么多，我们挑几个学习一下，`health`作为默认开放的端点之一，还是需要好好了解一下的。

### 设置何时显示信息

你可以使用得到的健康信息去检查你的应用程序，暴露的health信息依赖于下面两个属性配置：

```plain
management.endpoint.health.show-details
management.endpoint.health.show-components
```

这两个属性可以设置的值如下：

| Name              | Description                                           |
| :---------------- | :---------------------------------------------------- |
| `never`           | 永远不显示                                            |
| `when-authorized` | 需要授权，通过 `management.endpoint.health.roles`配置 |
| `always`          | 对所有用户显示                                        |

health端点通过健康指示器`HealthIndicator`获取不同的资源的健康信息，且Autuator内置了多个HealthIndicator的实现，太多啦太多啦，如果需要可以看看官网 [Auto-configured HealthIndicators](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-health-indicators)。

另外，如果你想启用或者禁用指定的指示器，你可以配置`management.health.key.enabled`，key需要被替换，官网表格上有。

如果你想设置全部的，你可以配置`management.health.defaults.enabled`

### 设置顺序

你可以通过以下设置显示顺序：

```yml
management:
  endpoint:
    health:
      status:
        order: "fatal,down,out-of-service,unknown,up"
```

### 设置响应码

响应码反应了总体健康状态。默认情况下，`OUT_OF_SERVICE`和`DOWN`的映射为503，其他状态为200。你可以按照下面的方式设置映射：

```yml
management:
  endpoint:
    health:
      status:
        http-mapping:
          down: 503
          fatal: 503
          out-of-service: 503
```

### 自定义健康信息

纵使，Actuator已经提供了许多内置实现，总会满足不了我们的需求，那如何去自定义呢？

- 注册实现HealthIndicator接口的Spring Bean。
- 提供health()方法的实现并返回健康响应，包括状态和其他需要显示的详细信息。

以下代码展示了一个案例：

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import java.util.Random;

@Component
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        // perform some specific health check
        boolean check = check();
        if (!check) {
            return Health.down().withDetail("Error Code", 0).build();
        }
        return Health.up().build();
    }

    private boolean check() {
        return new Random().nextBoolean();
    }

}
```

> id就是bena的名称去掉HealthIndicator后缀，这里id就是my。

看看相关的yml怎么配置？

```yml
management:
  endpoints:
    web:
      exposure:
        include: '*' # 需要开放的端点。默认值只打开 health 和 info 两个端点。通过设置 * ，可以开放所有端点。
    enabled-by-default: false
  endpoint:
    health:
      enabled: true
      show-details: always # 何时显示完整的健康信息
      show-components: always
      status:
        http-mapping: # 设置不同健康状态对应的响应状态码
          DOWN: 503
        order: FATAL, DOWN, OUT_OF_SERVICE, UP, UNKNOWN # 状态排序
```

测试一下，访问：`/actuator/health`，可以尝试多点几次，会出现UP和DOWN的情况，得到以下信息：

```json
{
  status: "DOWN",
  components: {
    diskSpace: {
      status: "UP",
      details: {
        total: 267117391872,
        free: 130840469504,
        threshold: 10485760,
        exists: true
      }
    },
    my: {
      status: "DOWN",
      details: {
        Error Code: 0
      }
    },
    ping: {
      status: "UP"
    }
  }
}
```

- my对应的就是我们自定义的MyHealthIndicator，里面详细的信息有：状态status，信息details。
- diskSpace对应DiskSpaceHealthIndicator，ping对应的是PingHealthIndicator。

其他的端点，感兴趣的小伙伴可以去到官网一一测试，总之，通过这些Actuator提供的端点，我们很容易就能够监控管理我们的应用程序。

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：https://gitee.com/tqbx/springboot-samples-learn。

## 参考阅读

- [Spring Boot Actuator: Production-ready Features](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready)
- [SpringBoot重点详解--使用Actuator进行健康监控](https://blog.csdn.net/pengjunlee/article/details/80235390)