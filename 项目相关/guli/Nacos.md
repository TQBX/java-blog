# Nacos启动和安装

官网：[https://nacos.io/en-us/](https://nacos.io/en-us/)

下载地址：[https://github.com/alibaba/nacos/releases/tag/1.3.1](https://github.com/alibaba/nacos/releases/tag/1.3.1)

Windows下载zip包之后，解压，进入bin目录直接双击startup.cmd运行。

访问[http://localhost:8848/nacos](http://localhost:8848/nacos)，进入登录页面，账号密码默认都是nacos。

# 服务注册流程

在service父模块中引入依赖

```xml
<!-- 服务注册 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

为服务配置server-addr

```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848 # nacos启动在8848端口
```

在服务启动类上加上注解`@EnableDiscoveryClient`

```java
@EnableDiscoveryClient
@SpringBootApplication
public class VodApplication {
    public static void main(String[] args) {
        SpringApplication.run(VodApplication.class);
    }

}
```

启动服务即可，就可以通过nacos管理页面的服务列表中看到服务已注册，且服务名为applicaiton.yml中配置的applicaiton.name。

# 服务调用

在service模块中引入依赖

```xml
<!--服务调用-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

在调用端服务启动类添加注解`@EnableFeignClients`

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class EduApplication {

    public static void main(String[] args) {
        SpringApplication.run(EduApplication.class,args);
    }

}
```

在调用端创建接口Interface，注解指定调用服务的名称，定义调用方法和路径

```java
@FeignClient("service-vod")//指定从哪个服务中调用功能,名称需要与被调用的服务名保持一致
@Component //防止在其他位置注入VodClient时IDEA报错
public interface VodClient {

    //定义调用方法的全路径
    @DeleteMapping("/eduVod/video/{videoSourceId}")
    public Result removeVideo(@PathVariable("videoSourceId") String videoSourceId);//PathVariable需要指定参数名称

}
```

在调用端注入VodClient，完成service-edu对service-vod的调用。

```java
@RestController
@RequestMapping("/eduService/video")
@CrossOrigin
public class EduVideoController {
    @Resource
    private EduVideoService videoService;

    /**
     * 注入VodClient
     */
    @Resource
    private VodClient vodClient;
    
    @DeleteMapping("/{id}")
    public Result delete(@PathVariable("id") String id){
        //根据id获取视频id
        EduVideo video = videoService.getById(id);
        if(!StringUtils.isEmpty(video.getVideoSourceId())){
            vodClient.removeVideo(video.getVideoSourceId());
        }
        videoService.removeById(id);
        return Result.ok();
    }
}
```

# Hystrix熔断

添加熔断器依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

开启熔断机制

```yml
# 开启熔断机制
feign:
  httpclient:
    enabled: true
```

创建interface实现类

```java
/**
 * 在调用服务失败,执行里面定义的方法
 *
 * @author Summerday
 */
@Component
public class VodClientImpl implements VodClient {

    @Override
    public Result removeVideo(String videoSourceId) {
        return Result.error().message("删除视频失败");
    }

    @Override
    public Result removeVideoList(List<String> videoIdList) {
        return Result.error().message("删除视频失败");
    }
}
```

在client接口@FeignClient注解上加上fallback属性，指定实现类。

```java
`@FeignClient(value = "service-vod",fallback = VodClientImpl.class)`
```

# Ribbon负载均衡

