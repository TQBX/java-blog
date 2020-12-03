# SpringBoot的定时任务与异步任务

**定时任务调度**功能在我们的开发中是非常常见的，随便举几个例子：定时清除一些过期的数据，定时发送邮件等等，实现定时任务调度的方式也十分多样，本篇文章主要学习各种实现定时任务调度方式的优缺点，以便为日后选择的时候提供一定的参考。

## 本篇要点

- 介绍Timer实现定时任务。
- 介绍ScheduledExecutorService实现定时任务。

- 介绍SpringBoot使用SpringTask实现定时任务。
- 介绍SpringBoot使用SpringTask实现异步任务。

## Timer实现定时任务

基于JDK自带的`java.util.Timer`，通过调度`java.util.TimeTask`让某一段程序按某一固定间隔，在某一延时之后定时执行。

缺点：

1. 无法指定某一时间的时候执行。
2. 存在潜在bug，Timer运行多个TimeTask时，只要其中之一没有捕获抛出的异常，其它任务便会自动终止运行。

```java
public class DemoTimer {

    //延时时间
    private static final long DELAY = 3000;

    //间隔时间
    private static final long PERIOD = 5000;

    public static void main(String[] args) {
        // 定义要执行的任务
        TimerTask task = new TimerTask() {
            @Override
            public void run() {
                System.out.println("任务执行 --> " + LocalDateTime.now());
            }
        };
        Timer timer = new Timer();
        timer.schedule(task, DELAY, PERIOD);
    }
}
```

### ScheduledExecutorService实现定时任务

阿里巴巴开发规范明确规定：希望开发者使用ScheduledExecutorService代替Timer。

>  多线程并行处理定时任务时，Timer运行多个TimeTask时，只要其中之一没有捕获抛出的异常，其它任务便会自动终止运行，使用ScheduledExecutorService则没有这个问题。

```java
public class DemoScheduledExecutorService {

    //延时时间
    private static final long DELAY = 3000;

    //间隔时间
    private static final long PERIOD = 5000;

    public static void main(String[] args) {

        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("任务执行 --> " + LocalDateTime.now());
            }
        };

        ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();
        service.scheduleAtFixedRate(task, DELAY, PERIOD, TimeUnit.MILLISECONDS);

    }
}
```

## SpringBoot使用Spring Task实现定时任务

### 自动配置实现原理

Spring为我们提供了异步执行任务调度的方式，提供TaskExecutor，TaskScheduler接口，而SpringBoot的自动配置类`org.springframework.boot.autoconfigure.task.TaskSchedulingAutoConfiguration`为我们默认注入了他们的实现：`ThreadPoolTaskScheduler`，本质上是`ScheduledExecutorService` 的封装，增强在调度时间上的功能。

![image-20201130223714127](img/Springboot%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1/image-20201130223714127.png)

```java
@ConditionalOnClass(ThreadPoolTaskScheduler.class)
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(TaskSchedulingProperties.class)
@AutoConfigureAfter(TaskExecutionAutoConfiguration.class)
public class TaskSchedulingAutoConfiguration {

	@Bean
	@ConditionalOnBean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
	@ConditionalOnMissingBean({ SchedulingConfigurer.class, TaskScheduler.class, ScheduledExecutorService.class })
	public ThreadPoolTaskScheduler taskScheduler(TaskSchedulerBuilder builder) {
		return builder.build();
	}

	@Bean
	@ConditionalOnMissingBean
	public TaskSchedulerBuilder taskSchedulerBuilder(TaskSchedulingProperties properties,
			ObjectProvider<TaskSchedulerCustomizer> taskSchedulerCustomizers) {
		TaskSchedulerBuilder builder = new TaskSchedulerBuilder();
		builder = builder.poolSize(properties.getPool().getSize());
		Shutdown shutdown = properties.getShutdown();
		builder = builder.awaitTermination(shutdown.isAwaitTermination());
		builder = builder.awaitTerminationPeriod(shutdown.getAwaitTerminationPeriod());
		builder = builder.threadNamePrefix(properties.getThreadNamePrefix());
		builder = builder.customizers(taskSchedulerCustomizers);
		return builder;
	}

}
```

### 新建工程，引入依赖

Spring Task是Spring Framework中的模块，我们只需引入`spring-boot-starter`依赖就可以了。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
```

### 编写配置类@EnableScheduling

```java
@Configuration
@EnableScheduling
public class ScheduleConfig {

}
```

- `@Configuration`表明这是个配置类。
- `@EnableScheduling`表明启用Spring的定时任务调度功能。

### 定义定时任务@Scheduled

```java
@Component
@Slf4j
public class DemoTask {

    private final AtomicInteger counts = new AtomicInteger();

    @Scheduled(cron = "0/5 * * * * *")
    public void execute() {
        log.info("[定时任务第 {} 次执行]", counts.incrementAndGet());
    }
}
```

- `@Component`表明该类需要被扫描，以便于Spring容器管理。
- `@Scheduled`标注**需要调度执行的方法**，定义执行规则，其必须指定`cron`、`fixedDelay`或`fixedRate`三个属性其中一个。
  - `cron`：定义Spring cron表达式，网上有在线cron生成器，可以对照着编写符合需求的定时任务。
  - `fixedDelay` ：固定执行间隔，单位：毫秒。注意，以调用**完成时刻**为开始计时时间。
  - `fixedRate` ：固定执行间隔，单位：毫秒。注意，以调用**开始时刻**为开始计时时间。

### 主启动类

```java
@SpringBootApplication
public class SpringBootTaskApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootTaskApplication.class, args);
    }
}

```

### 定义配置文件

Spring Task 调度任务的配置，对应 TaskSchedulingProperties 配置类。SpringBoot允许我们在yml或properties定制这些外部化配置，如果不配置也是没有关系的，自动配置已经给你一套默认的值了。

```yml
spring:
  task:
    scheduling:
      thread-name-prefix: summerday- # 线程池的线程名的前缀。默认为 scheduling- ，建议根据自己应用来设置
      pool:
        size: 10 # 线程池大小。默认为 1 ，根据自己应用来设置
      shutdown:
        await-termination: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true
        await-termination-period: 60 # 等待任务完成的最大时长，单位为秒。默认为 0 ，根据自己应用来设置
```

### 启动项目测试

```java
# 初始化一个 ThreadPoolTaskScheduler 任务调度器
2020-11-30 23:04:51.886  INFO 10936 --- [  restartedMain] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService 'taskScheduler'
# 每5s执行一次定时任务
2020-11-30 23:04:55.002  INFO 10936 --- [    summerday-1] com.hyh.task.DemoTask                    : [定时任务第 1 次执行]
2020-11-30 23:05:00.002  INFO 10936 --- [    summerday-1] com.hyh.task.DemoTask                    : [定时任务第 2 次执行]
2020-11-30 23:05:05.002  INFO 10936 --- [    summerday-2] com.hyh.task.DemoTask                    : [定时任务第 3 次执行]
2020-11-30 23:05:10.001  INFO 10936 --- [    summerday-1] com.hyh.task.DemoTask                    : [定时任务第 4 次执行]
2020-11-30 23:05:15.002  INFO 10936 --- [    summerday-3] com.hyh.task.DemoTask                    : [定时任务第 5 次执行]
```

## SpringTask异步任务

SpringTask除了`@Scheduled、@EnableScheduling`同步定时任务之外，还有`@Async、@EnableAsync` 开启异步的定时任务调度。

SpringBoot自动配置类对异步的支持：`org.springframework.boot.autoconfigure.task.TaskExecutionAutoConfiguration`

### @Async注解添加

```java
    @Async
    @Scheduled(cron = "0/1 * * * * *")
    public void asyncTask() {
        sleep();
        System.out.println(Thread.currentThread().getName() + " async-task 执行,当前时间: " + LocalDateTime.now());
    }
```

### @EnableAsync注解添加

```java
@Configuration
@EnableScheduling  // 同步
@EnableAsync // 异步
public class ScheduleConfig {

}
```

### 配置文件

```yml
spring:
  task:
    # Spring 执行器配置，对应 TaskExecutionProperties 配置类。对于 Spring 异步任务，会使用该执行器。
    execution:
      thread-name-prefix: async- # 线程池的线程名的前缀。默认为 task- ，建议根据自己应用来设置
      pool: # 线程池相关
        core-size: 8 # 核心线程数，线程池创建时候初始化的线程数。默认为 8 。
        max-size: 20 # 最大线程数，线程池最大的线程数，只有在缓冲队列满了之后，才会申请超过核心线程数的线程。默认为 Integer.MAX_VALUE
        keep-alive: 60s # 允许线程的空闲时间，当超过了核心线程之外的线程，在空闲时间到达之后会被销毁。默认为 60 秒
        queue-capacity: 200 # 缓冲队列大小，用来缓冲执行任务的队列的大小。默认为 Integer.MAX_VALUE 。
        allow-core-thread-timeout: true # 是否允许核心线程超时，即开启线程池的动态增长和缩小。默认为 true 。
      shutdown:
        await-termination: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true
        await-termination-period: 60 # 等待任务完成的最大时长，单位为秒。默认为 0 ，根据自己应用来设置
```

## 同步与异步对比

```java
@Component
public class DemoAsyncTask {

    @Scheduled(cron = "0/1 * * * * *")
    public void synTask() {
        sleep();
        System.out.println(Thread.currentThread().getName() + " syn-task 执行,当前时间: " + LocalDateTime.now());
    }

    @Async
    @Scheduled(cron = "0/1 * * * * *")
    public void asyncTask() {
        sleep();
        System.out.println(Thread.currentThread().getName() + " async-task 执行,当前时间: " + LocalDateTime.now());
    }

    private void sleep() {
        try {
            Thread.sleep(10 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

同时开启同步和异步任务，假设任务本身耗时较长，且间隔较短：间隔1s，执行10s，同步与异步执行的差异就此体现。

![image-20201201213140127](img/Springboot%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1/image-20201201213140127.png)

可以看到，同步任务并没有每间隔1s就执行，而是串行在一起，等前一个任务执行完才执行。而异步任务则不一样，成功将串行化的任务并行化。

## 五、源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [芋道 Spring Boot 定时任务入门](http://www.iocoder.cn/Spring-Boot/Job/)
- [一起来学SpringBoot | 第十六篇：定时任务详解](http://blog.battcn.com/2018/05/29/springboot/v2-other-scheduling/)