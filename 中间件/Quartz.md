[toc]

## Quartz是什么

Quartz是一个功能强大的开源任务调度库，几乎可以集成到任何Java应用程序中，无论是超小型的独立应用还是超大型电子商务系统。

它常用于企业级应用中：

- Driving Process Workflow：当新订单下达，可以安排一个30分钟内触发的任务，检查订单状态。
- System Maintenance：安排每个工作日晚上11点将数据库内容转储到文件的任务。
- Providing reminder services：提供提醒服务。

Quartz还支持集群模式和对JTA事务。

## Quartz中的重要API及概念

> http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/

### 超重要API

- **Scheduler** 和调度程序交互的主要API
  - 生命周期从SchedulerFactoru创建它开始，到调用shutdown方法结束。
  - 一旦Scheduler创建，任何关于scheduling相关的事，他都为所欲为：添加、删除、列出所有的Jobs和triggers、暂停触发器等。
  - 在start方法之前，不会做任何事情。
- **Job** 你希望被调度器调度的任务组件接口。
  - 当Job的触发器触发时，调度程序的工作线程将调用execute方法。
  - 该方法接收一个`JobExecutionContext `对象，为Job实例提供了丰富的运行时环境信息，比如：scheduler、trigger、jobDataMap、job、calendar、各种time等。
- **JobDetail** 用于定义任务。
  - JobDetail对象由Quartz客户端在将job加入Scheduler提供，也就是你的程序。
  - 它包含了不同为job设置的属性，还有可以用来为job储存状态信息的JobDataMap。
  - 注意它和Job的区别，它实际上是Job实例的属性。【Job定义如何执行，JobDetail定义有何属性】
- **Trigger** 触发任务执行。
  - 触发器可能具有与之关联的JobDataMap，以便于将特定于触发器触发的参数传递给Job。
  - Quartz提供了几种不同的触发器，SimpleTrigger和CronTrigger比较常用。
  - 如果你需要一次性执行作业或需要在给定的时间触发一个作业并重复执行N次且有两次执行间有延时delay，SimpleTrigger较为方便。
  - 如果你希望基于类似日期表触发执行任务，CronTrgger推荐使用。
- **JobBuilder** 用于构建JobDetail的。
- **TriggerBuilder** 用于构建Trigger的。

> Quartz提供了各种各样的Builder类，定义了Domain Specific Language，且都提供了静态的创建方法，我们可以使用`import static`简化书写。

### 重要概念

- **Identity**
  - 当作业和触发器在Quartz调度程序中注册时，会获得标识键。
  - JobKey和TriggerKey允许被置入group中，易于组织管理。
  - 唯一的，是name和group的组合标识。
- **JobDataMap**
  - 是Map的实现，具有key-value相关操作，存储可序列化数据对象，供Job实例在执行时使用。
  - 可以使用`usingJobData(key,value)`在构建Jobdetail的时候传入数据，使用`jobDetail.getJobDataMap()`获取map。

## Quartz设计理念：为什么设计Job和Trigger？

>  While developing Quartz, <u>we decided that it made sense to create a separation between the schedule and the work to be performed on that schedule. This has (in our opinion) many benefits.</u>
>
> For example, Jobs can be created and stored in the job scheduler independent of a trigger, and many triggers can be associated with the same job. Another benefit of this loose-coupling is the ability to configure jobs that remain in the scheduler after their associated triggers have expired, so that that it can be rescheduled later, without having to re-define it. It also allows you to modify or replace a trigger without having to re-define its associated job.

隔离schedule和schedule上执行的Job，优点是可见的：

可以独立于触发器创建作业并将其存储在作业调度程序中，并且许多触发器可以与同一作业相关联。这样的松耦合好处是什么？

- 如果触发器过期，作业还可以保留在程序中，以便重新调度，而不必重新定义。
- 如果你希望修改替换某个触发器，你不必重新定义其关联的作业。

## 最简单的Quartz使用案例

导入依赖

```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.2</version>
</dependency>
```

简单案例如下

```java
public class QuartzTest {
	// 你需要在start和shutdown之间执行你的任务。
    public static void main(String[] args) {

        try {
            // 从工厂中获取Scheduler示例
            Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

            // 开始
            scheduler.start();

            // 定义Job，并将其绑定到HelloJob类中
            JobDetail job = JobBuilder.newJob(HelloJob.class)
                    .withIdentity("job1", "group1") // name 和 group
                    .usingJobData("username", "天乔巴夏") // 置入JobDataMap
                    .usingJobData("age", "20")
                    .withDescription("desc-demo")
                    .build();

            // 触发Job执行，每40s执行一次
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity("trigger1", "group1")
                    .startNow() // 立即开始
                    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
                            .withIntervalInSeconds(40)
                            .repeatForever())
                    .build();

            // 告诉 quartz 使用trigger来调度job
            scheduler.scheduleJob(job, trigger);
			// 关闭，线程终止
            scheduler.shutdown();

        } catch (SchedulerException se) {
            se.printStackTrace();
        }
    }

}

@Slf4j
@NoArgsConstructor
public class HelloJob implements Job {

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        // 从context中获取属性
        JobDetail jobDetail = context.getJobDetail();
        JobDataMap jobDataMap = jobDetail.getJobDataMap();
        JobKey key = jobDetail.getKey();
        Class<? extends Job> jobClass = jobDetail.getJobClass();
        String description = jobDetail.getDescription();

        String username = jobDataMap.getString("username");
        int age = jobDataMap.getIntValue("age");

        log.info("\nJobKey : {},\n JobClass : {},\n JobDesc : {},\n username : {},\n age : {}",
                key, jobClass.getName(), description, username, age);
    }
}
```

启动测试，打印日志如下：

```java
01:23:12.406 [main] INFO org.quartz.impl.StdSchedulerFactory - Using default implementation for ThreadExecutor
01:23:12.414 [main] INFO org.quartz.simpl.SimpleThreadPool - Job execution threads will use class loader of thread: main
01:23:12.430 [main] INFO org.quartz.core.SchedulerSignalerImpl - Initialized Scheduler Signaller of type: class org.quartz.core.SchedulerSignalerImpl
01:23:12.430 [main] INFO org.quartz.core.QuartzScheduler - Quartz Scheduler v.2.3.2 created.
01:23:12.432 [main] INFO org.quartz.simpl.RAMJobStore - RAMJobStore initialized.
01:23:12.433 [main] INFO org.quartz.core.QuartzScheduler - Scheduler meta-data: Quartz Scheduler (v2.3.2) 'DefaultQuartzScheduler' with instanceId 'NON_CLUSTERED'
  Scheduler class: 'org.quartz.core.QuartzScheduler' - running locally.
  NOT STARTED.
  Currently in standby mode.
  Number of jobs executed: 0
  Using thread pool 'org.quartz.simpl.SimpleThreadPool' - with 10 threads.
  Using job-store 'org.quartz.simpl.RAMJobStore' - which does not support persistence. and is not clustered.

01:23:12.433 [main] INFO org.quartz.impl.StdSchedulerFactory - Quartz scheduler 'DefaultQuartzScheduler' initialized from default resource file in Quartz package: 'quartz.properties'
01:23:12.433 [main] INFO org.quartz.impl.StdSchedulerFactory - Quartz scheduler version: 2.3.2
01:23:12.434 [main] INFO org.quartz.core.QuartzScheduler - Scheduler DefaultQuartzScheduler_$_NON_CLUSTERED started.
01:23:12.434 [DefaultQuartzScheduler_QuartzSchedulerThread] DEBUG org.quartz.core.QuartzSchedulerThread - batch acquisition of 0 triggers
01:23:12.443 [DefaultQuartzScheduler_QuartzSchedulerThread] DEBUG org.quartz.core.QuartzSchedulerThread - batch acquisition of 1 triggers
01:23:12.445 [DefaultQuartzScheduler_QuartzSchedulerThread] DEBUG org.quartz.simpl.PropertySettingJobFactory - Producing instance of Job 'group1.job1', class=com.hyhwky.HelloJob
01:23:12.451 [DefaultQuartzScheduler_QuartzSchedulerThread] DEBUG org.quartz.core.QuartzSchedulerThread - batch acquisition of 0 triggers
01:23:12.452 [DefaultQuartzScheduler_Worker-1] DEBUG org.quartz.core.JobRunShell - Calling execute on job group1.job1
01:23:12.452 [DefaultQuartzScheduler_Worker-1] INFO com.hyhwky.HelloJob - 
JobKey : group1.job1,
 JobClass : com.hyhwky.HelloJob,
 JobDesc : desc-demo,
 username : 天乔巴夏,
 age : 20
```

我们可以看到quartz已经被初始化了，初始化配置如下，在`org\quartz-scheduler\quartz\2.3.2\quartz-2.3.2.jar!\org\quartz\quartz.properties`

```properties
# 调度器配置
org.quartz.scheduler.instanceName: DefaultQuartzScheduler
org.quartz.scheduler.rmi.export: false
org.quartz.scheduler.rmi.proxy: false
org.quartz.scheduler.wrapJobExecutionInUserTransaction: false

# 线程池配置
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool 
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true

# 存储配置
org.quartz.jobStore.misfireThreshold: 60000 #trigger 容忍时间60s

org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
```

更多的配置：[Quartz Configuration Reference](http://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigMain.html)

## Job实例和JobDetail

Job和JobDetail

- Job是正在执行的作业实例，JobDetail是作业定义。
- 一个Job可以创建多个JobDetail，拥有不同的JobDataMap。

这样定义有什么好处呢？举个例子吧，假设现在你定义了一个类实现了Job接口，比如：SendEmailJob。如果你希望根据用户的姓名，选择指定的人发送，那么你可以通过JobDataMap绑定参数传递进JobDetail中，也就是说我们需要创建两个不同的JobDetail，比如：SendEmailToSummerday和SendEmailToTQBX，他们拥有各自独立的JobDataMap，实现更加灵活。

## Job的State和Concurrency

> https://blog.csdn.net/fly_captain/article/details/83029440

### @DisallowConcurrentExecution

该注解标注在Job类上，意思是不能并发执同一个JobDetail定义的多个实例，但可以同时执行多个不同的JobDetail定义的实例。

拿上面的例子继续举例，假设SendEmailJob标注了此注解，表明同一时间可以同时执行SendEmailToSummerday和SendEmailToTQBX，因为他们是不同的JobDetail，但是不能同时执行多个SendEmailToSummerday。

### @PersistJobDataAfterExecution

该注解也标注在Job类上，告诉Scheduler正常执行完Job之后，重新存储更新一下JobDataMap。一般标注此注解的Job类应当考虑也加上@DisallowConcurrentExecution注解，以避免同时执行Job时出现JobDataMap存储的竞争。

## Trigger常见使用

> 更多例子可以查看文档，非常详细：[SimpleTrigger](http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/tutorial-lesson-05.html) 和 [CronTrigger](http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/tutorial-lesson-06.html)

构建一个触发器，该触发器将立即触发，然后每五分钟重复一次，直到小时 22：00：

```java
 
import static org.quartz.TriggerBuilder.*;
import static org.quartz.SimpleScheduleBuilder.*;
import static org.quartz.DateBuilder.*: 

 SimpleTrigger trigger = (SimpleTrigger) newTrigger()
    .withIdentity("trigger7", "group1")
    .withSchedule(simpleSchedule()
        .withIntervalInMinutes(5)
        .repeatForever())
    .endAt(dateOf(22, 0, 0))
    .build();
```

构建一个触发器，该触发器将在周三上午 10：42 触发，在系统默认值以外的时区中：

```java
import static org.quartz.TriggerBuilder.*;
import static org.quartz.CronScheduleBuilder.*;
import static org.quartz.DateBuilder.*:

 trigger = newTrigger()
    .withIdentity("trigger3", "group1")
    .withSchedule(cronSchedule("0 42 10 ? * WED")) // [秒] [分] [时] [月的第几天] [月] [一星期的第几天] [年(可选)]
    .inTimeZone(TimeZone.getTimeZone("America/Los_Angeles"))
    .forJob(myJobKey)
    .build();
```

## Quartz存储与持久化

> Job的持久化是非常重要的，如果Job不能持久化，一旦不再有与之关联的Trigger，他就会自动从调度程序中被删除。

Job存储器的类型：

| 类型          | 优点                                                         | 缺点                                                         |
| :------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| RAMJobStore   | 不要外部数据库，配置容易，运行速度快                         | 因为调度程序信息是存储在被分配给 JVM 的内存里面，所以，当应用程序停止运行时，所有调度信息将被丢失。另外因为存储到JVM内存里面，所以可以存储多少个 Job 和 Trigger 将会受到限制 |
| JDBC 作业存储 | 支持集群，因为所有的任务信息都会保存到数据库中，可以控制事物，还有就是如果应用服务器关闭或者重启，任务信息都不会丢失，并且可以恢复因服务器关闭或者重启而导致执行失败的任务 | 运行速度的快慢取决与连接数据库的快慢                         |

具体的使用方法：[http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/tutorial-lesson-09.html](http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/tutorial-lesson-09.html)

## SpringBoot整合Quartz单机版快速开始

学习完非Spring环境下Quartz的使用，再来看SpringBoot你会感到更加自如，因为SpringBoot无非是利用它自动配置的特性，将一些重要的Bean自动配置到环境中，我们直接开箱即用，关于Quartz的自动配置定义在QuartzAutoConfiguration中。

### 引入依赖

主要是`spring-boot-starter-quartz`这个依赖，是SpringBoot与Quartz的整合。

```xml
        <!-- 实现对 Spring MVC 的自动化配置 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 实现对 Quartz 的自动化配置 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-quartz</artifactId>
        </dependency>
```

### 创建Job

```java
@Slf4j
public class FirstJob extends QuartzJobBean {

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        String now = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss").format(LocalDateTime.now());
        log.info("当前的时间: " + now);
    }
}
```

### 调度器Scheduler绑定

#### 自动配置，这里演示SimpleScheduleBuilder

```java
@Configuration
public class QuartzConfig {

    private static final String ID = "SUMMERDAY";

    @Bean
    public JobDetail jobDetail1() {
        return JobBuilder.newJob(FirstJob.class)
                .withIdentity(ID + " 01")
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger trigger1() {
        // 简单的调度计划的构造器
        SimpleScheduleBuilder scheduleBuilder = SimpleScheduleBuilder.simpleSchedule()
                .withIntervalInSeconds(5) // 频率
                .repeatForever(); // 次数

        return TriggerBuilder.newTrigger()
                .forJob(jobDetail1())
                .withIdentity(ID + " 01Trigger")
                .withSchedule(scheduleBuilder)
                .build();
    }
}
```

#### 手动配置，这里演示CronScheduleBuilder

```java
@Component
public class JobInit implements ApplicationRunner {

    private static final String ID = "SUMMERDAY";

    @Autowired
    private Scheduler scheduler;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        JobDetail jobDetail = JobBuilder.newJob(FirstJob.class)
                .withIdentity(ID + " 01")
                .storeDurably()
                .build();
        CronScheduleBuilder scheduleBuilder =
                CronScheduleBuilder.cronSchedule("0/5 * * * * ? *");
        // 创建任务触发器
        Trigger trigger = TriggerBuilder.newTrigger()
                .forJob(jobDetail)
                .withIdentity(ID + " 01Trigger")
                .withSchedule(scheduleBuilder)
                .startNow() //立即執行一次任務
                .build();
        // 手动将触发器与任务绑定到调度器内
        scheduler.scheduleJob(jobDetail, trigger);
    }
}

```

### yml配置

```yml
spring:
  # Quartz 的配置，对应 QuartzProperties 配置类
  quartz:
    job-store-type: memory # Job 存储器类型。默认为 memory 表示内存，可选 jdbc 使用数据库。
    auto-startup: true # Quartz 是否自动启动
    startup-delay: 0 # 延迟 N 秒启动
    wait-for-jobs-to-complete-on-shutdown: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true
    overwrite-existing-jobs: false # 是否覆盖已有 Job 的配置
    properties: # 添加 Quartz Scheduler 附加属性
      org:
        quartz:
          threadPool:
            threadCount: 25 # 线程池大小。默认为 10 。
            threadPriority: 5 # 线程优先级
            class: org.quartz.simpl.SimpleThreadPool # 线程池类型
#    jdbc: # 这里暂时不说明，使用 JDBC 的 JobStore 的时候，才需要配置
```

SpringBoot自动配置中：`spring.quartz`对应的配置项定义在`QuartzProperties`中。

### 主启动类

```java
@SpringBootApplication
public class DemoSpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoSpringBootApplication.class, args);
    }
}
```

### 测试

启动程序，job1每5s执行一次，打印当前时间。z

```java
2020-12-24 16:54:25.575  INFO 6352 --- [eduler_Worker-1] com.hyhwky.task.config.FirstJob          : 当前的时间: 2020-12-24 16:54:25
2020-12-24 16:54:30.262  INFO 6352 --- [eduler_Worker-2] com.hyhwky.task.config.FirstJob          : 当前的时间: 2020-12-24 16:54:30
2020-12-24 16:54:35.263  INFO 6352 --- [eduler_Worker-3] com.hyhwky.task.config.FirstJob          : 当前的时间: 2020-12-24 16:54:35
2020-12-24 16:54:40.263  INFO 6352 --- [eduler_Worker-4] com.hyhwky.task.config.FirstJob          : 当前的时间: 2020-12-24 16:54:40
2020-12-24 16:54:45.262  INFO 6352 --- [eduler_Worker-5] com.hyhwky.task.config.FirstJob          : 当前的时间: 2020-12-24 16:54:45
```

## QuartzJobBean

QuartzJobBean实现了Job，并且定义了公用的execute方法，子类可以继承QuartzJobBean并实现executeInternal方法。

```java
public abstract class QuartzJobBean implements Job {

	/**
	 * This implementation applies the passed-in job data map as bean property
	 * values, and delegates to {@code executeInternal} afterwards.
	 * @see #executeInternal
	 */
	@Override
	public final void execute(JobExecutionContext context) throws JobExecutionException {
		try {
             // 将当前对象包装为BeanWrapper
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            // 设置属性
			MutablePropertyValues pvs = new MutablePropertyValues();
			pvs.addPropertyValues(context.getScheduler().getContext());
			pvs.addPropertyValues(context.getMergedJobDataMap());
			bw.setPropertyValues(pvs, true);
		}
		catch (SchedulerException ex) {
			throw new JobExecutionException(ex);
		}
        // 子类实现该方法
		executeInternal(context);
	}

	/**
	 * Execute the actual job. The job data map will already have been
	 * applied as bean property values by execute. The contract is
	 * exactly the same as for the standard Quartz execute method.
	 * @see #execute
	 */
	protected abstract void executeInternal(JobExecutionContext context) throws JobExecutionException;

}
```

## 参考阅读

- [【Quartz】Quartz存储与持久化-基于quartz.properties的配置](http://blog.csdn.net/evankaka)

- [芋道 Spring Boot 定时任务入门](http://www.iocoder.cn/Spring-Boot/Job/)
- https://www.jianshu.com/p/7663f0ed486a

- [WIKIPEDIA ： Quartz(scheduler)](https://en.wikipedia.org/wiki/Quartz_(scheduler))
- http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/