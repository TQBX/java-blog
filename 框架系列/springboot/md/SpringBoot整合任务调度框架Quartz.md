[toc]

> 本文侧重SpringBoot与Quartz的整合，Quartz的基本入门概念不清楚的小伙伴可以看看这篇文章：[任务调度框架Quartz快速入门！](https://www.cnblogs.com/summerday152/p/14192845.html)

## 本篇要点

- 介绍SpringBoot与Quartz单机版整合。
- 介绍Quartz持久化存储。

## SpringBoot与Quartz单机版快速整合

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

为了演示两种Trigger及两种配置方式，我们创建两个不同的Job。

```java
@Slf4j
public class FirstJob extends QuartzJobBean {

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        String now = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss").format(LocalDateTime.now());
        log.info("当前的时间: " + now);
    }
}

@Slf4j
public class SecondJob extends QuartzJobBean {

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        String now = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss").format(LocalDateTime.now());
        log.info("SecondJob执行, 当前的时间: " + now);
    }
}
```

我们在创建Job的时候，可以实现Job接口，也可以继承QuartzJobBean。

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

### 调度器Scheduler绑定

Scheduler绑定有两种方式，一种是使用bena的自动配置，一种是Scheduler手动配置。

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

启动程序，FirstJob每5s执行一次，SecondJob每10s执行一次。

```java
 [eduler_Worker-1] com.hyhwky.standalone.FirstJob           : FirstJob执行, 当前的时间: 2020-12-26 16:54:00
 [eduler_Worker-2] com.hyhwky.standalone.SecondJob          : SecondJob执行, 当前的时间: 2020-12-26 16:54:00
 [eduler_Worker-3] com.hyhwky.standalone.FirstJob           : FirstJob执行, 当前的时间: 2020-12-26 16:54:05
 [eduler_Worker-4] com.hyhwky.standalone.SecondJob          : SecondJob执行, 当前的时间: 2020-12-26 16:54:10
 [eduler_Worker-5] com.hyhwky.standalone.FirstJob           : FirstJob执行, 当前的时间: 2020-12-26 16:54:10
 [eduler_Worker-6] com.hyhwky.standalone.FirstJob           : FirstJob执行, 当前的时间: 2020-12-26 16:54:15
 [eduler_Worker-7] com.hyhwky.standalone.SecondJob          : SecondJob执行, 当前的时间: 2020-12-26 16:54:20
```

## Quartz持久化配置

Quartz持久化配置提供了两种存储器：

| 类型          | 优点                                                         | 缺点                                                         |
| :------------ | :----------------------------------------------------------- | :----------------------------------------------------------- |
| RAMJobStore   | 不要外部数据库，配置容易，运行速度快                         | 因为调度程序信息是存储在被分配给 JVM 的内存里面，所以，当应用程序停止运行时，所有调度信息将被丢失。另外因为存储到JVM内存里面，所以可以存储多少个 Job 和 Trigger 将会受到限制 |
| JDBC 作业存储 | 支持集群，因为所有的任务信息都会保存到数据库中，可以控制事物，还有就是如果应用服务器关闭或者重启，任务信息都不会丢失，并且可以恢复因服务器关闭或者重启而导致执行失败的任务 | 运行速度的快慢取决与连接数据库的快慢                         |

### 创建数据库表

为了测试Quartz的持久化配置，我们事先在mysql中创建一个数据库quartz，并执行脚本，脚本藏在`org\quartz-scheduler\quartz\2.3.2\quartz-2.3.2.jar!\org\quartz\impl\jdbcjobstore\tables_mysql_innodb.sql`，jdbcjobstore中有支持许多种数据库的脚本，可以按需执行。

```sql
mysql> use quartz;
Database changed
mysql> show tables;
+--------------------------+
| Tables_in_quartz         |
+--------------------------+
| qrtz_blob_triggers       |## blog类型存储triggers
| qrtz_calendars           |## 以blog类型存储Calendar信息
| qrtz_cron_triggers       |## 存储cron trigger信息
| qrtz_fired_triggers      |## 存储已触发的trigger相关信息
| qrtz_job_details         |## 存储每一个已配置的job details
| qrtz_locks               |## 存储悲观锁的信息
| qrtz_paused_trigger_grps |## 存储已暂停的trigger组信息
| qrtz_scheduler_state     |## 存储Scheduler状态信息
| qrtz_simple_triggers     |## 存储simple trigger信息
| qrtz_simprop_triggers    |## 存储其他几种trigger信息
| qrtz_triggers            |## 存储已配置的trigger信息
+--------------------------+
```

所有的表中都含有一个`SCHED_NAME`字段，对应我们配置的`scheduler-name`，相同 Scheduler-name的节点，形成一个 Quartz 集群。

### 引入mysql相关依赖

```xml
    <dependencies>
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

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
    </dependencies>
```

### 配置yml

```yml
spring:
  datasource:
    quartz:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/quartz?serverTimezone=GMT%2B8
      username: root
      password: 123456
  quartz:
    job-store-type: jdbc # 使用数据库存储
    scheduler-name: hyhScheduler # 相同 Scheduler 名字的节点，形成一个 Quartz 集群
    wait-for-jobs-to-complete-on-shutdown: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true
    jdbc:
      initialize-schema: never # 是否自动使用 SQL 初始化 Quartz 表结构。这里设置成 never ，我们手动创建表结构。
    properties:
      org:
        quartz:
          # JobStore 相关配置
          jobStore:
            dataSource: quartzDataSource # 使用的数据源
            class: org.quartz.impl.jdbcjobstore.JobStoreTX # JobStore 实现类
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            tablePrefix: QRTZ_ # Quartz 表前缀
            isClustered: true # 是集群模式
            clusterCheckinInterval: 1000
            useProperties: false
          # 线程池相关配置
          threadPool:
            threadCount: 25 # 线程池大小。默认为 10 。
            threadPriority: 5 # 线程优先级
            class: org.quartz.simpl.SimpleThreadPool # 线程池类型
```

### 配置数据源

```java
@Configuration
public class DataSourceConfiguration {

    private static HikariDataSource createHikariDataSource(DataSourceProperties properties) {
        // 创建 HikariDataSource 对象
        HikariDataSource dataSource = properties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
        // 设置线程池名
        if (StringUtils.hasText(properties.getName())) {
            dataSource.setPoolName(properties.getName());
        }
        return dataSource;
    }

    /**
     * 创建 quartz 数据源的配置对象
     */
    @Primary
    @Bean(name = "quartzDataSourceProperties")
    @ConfigurationProperties(prefix = "spring.datasource.quartz")
    // 读取 spring.datasource.quartz 配置到 DataSourceProperties 对象
    public DataSourceProperties quartzDataSourceProperties() {
        return new DataSourceProperties();
    }

    /**
     * 创建 quartz 数据源
     */
    @Bean(name = "quartzDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.quartz.hikari")
    @QuartzDataSource
    public DataSource quartzDataSource() {
        // 获得 DataSourceProperties 对象
        DataSourceProperties properties = this.quartzDataSourceProperties();
        // 创建 HikariDataSource 对象
        return createHikariDataSource(properties);
    }

}
```

### 创建任务

```java
@Component
public class JobInit implements ApplicationRunner {

    private static final String ID = "SUMMERDAY";

    @Autowired
    private Scheduler scheduler;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        JobDetail jobDetail = JobBuilder.newJob(SecondJob.class)
                .withIdentity(ID + " 02")
                .storeDurably()
                .build();
        CronScheduleBuilder scheduleBuilder =
                CronScheduleBuilder.cronSchedule("0/10 * * * * ? *");
        // 创建任务触发器
        Trigger trigger = TriggerBuilder.newTrigger()
                .forJob(jobDetail)
                .withIdentity(ID + " 02Trigger")
                .withSchedule(scheduleBuilder)
                .startNow() //立即執行一次任務
                .build();
        Set<Trigger> set = new HashSet<>();
        set.add(trigger);
        // boolean replace 表示启动时对数据库中的quartz的任务进行覆盖。
        scheduler.scheduleJob(jobDetail, set, true);
    }
}
```

### 启动测试

启动测试之后，我们的quartz任务相关信息就已经成功存储到mysql中了。

```sql
mysql> select * from qrtz_simple_triggers;
+--------------+---------------------+---------------+--------------+-----------------+-----------------+
| SCHED_NAME   | TRIGGER_NAME        | TRIGGER_GROUP | REPEAT_COUNT | REPEAT_INTERVAL | TIMES_TRIGGERED |
+--------------+---------------------+---------------+--------------+-----------------+-----------------+
| hyhScheduler | SUMMERDAY 01Trigger | DEFAULT       |           -1 |            5000 |             812 |
+--------------+---------------------+---------------+--------------+-----------------+-----------------+
1 row in set (0.00 sec)

mysql> select * from qrtz_cron_triggers;
+--------------+---------------------+---------------+------------------+---------------+
| SCHED_NAME   | TRIGGER_NAME        | TRIGGER_GROUP | CRON_EXPRESSION  | TIME_ZONE_ID  |
+--------------+---------------------+---------------+------------------+---------------+
| hyhScheduler | SUMMERDAY 02Trigger | DEFAULT       | 0/10 * * * * ? * | Asia/Shanghai |
+--------------+---------------------+---------------+------------------+---------------+
```

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [【Quartz】Quartz存储与持久化-基于quartz.properties的配置](http://blog.csdn.net/evankaka)
- [芋道 Spring Boot 定时任务入门](http://www.iocoder.cn/Spring-Boot/Job/)
- https://www.jianshu.com/p/7663f0ed486a
- [WIKIPEDIA ： Quartz(scheduler)](https://en.wikipedia.org/wiki/Quartz_(scheduler))
- http://www.quartz-scheduler.org/documentation/quartz-2.2.2/tutorials/
- [Quartz数据库表分析](https://blog.csdn.net/xiaoniu_888/article/details/83181078)