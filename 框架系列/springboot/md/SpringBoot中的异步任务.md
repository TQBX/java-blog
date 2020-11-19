- [1. 概述](http://www.iocoder.cn/Spring-Boot/Async-Job/)
- [2. 快速入门](http://www.iocoder.cn/Spring-Boot/Async-Job/)
- [3. 异步回调](http://www.iocoder.cn/Spring-Boot/Async-Job/)
- [4. 异步异常处理器](http://www.iocoder.cn/Spring-Boot/Async-Job/)
- [5. 自定义执行器](http://www.iocoder.cn/Spring-Boot/Async-Job/)
- [666. 彩蛋](http://www.iocoder.cn/Spring-Boot/Async-Job/)

------

------

> 本文在提供完整代码示例，可见 https://github.com/YunaiV/SpringBoot-Labs 的 [lab-29](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29) 目录。
>
> 原创不易，给点个 [Star](https://github.com/YunaiV/SpringBoot-Labs/stargazers) 嘿，一起冲鸭！

# 1. 概述

在日常开发中，我们的逻辑都是**同步调用**，顺序执行。在一些场景下，我们会希望异步调用，将和主线程关联度低的逻辑**异步调用**，以实现让主线程更快的执行完成，提升性能。例如说：记录用户访问日志到数据库，记录管理员操作日志到数据库中。

> 异步调用，对应的是同步调用。
>
> - 同步调用：指程序按照 定义顺序 依次执行，每一行程序都必须等待上一行程序执行完成之后才能执行；
> - 异步调用：指程序在顺序执行时，不等待异步调用的语句返回结果，就执行后面的程序。

考虑到异步调用的**可靠性**，我们一般会考虑引入分布式消息队列，例如说 RabbitMQ、RocketMQ、Kafka 等等。但是在一些时候，我们并不需要这么高的可靠性，可以使用**进程内**的队列或者线程池。例如说示例代码如下：

```
public class Demo {    public static void main(String[] args) {        // 创建线程池。这里只是临时测试，不要扣艿艿遵守阿里 Java 开发规范，YEAH        ExecutorService executor = Executors.newFixedThreadPool(10);        // 提交任务到线程池中执行。        executor.submit(new Runnable() {            @Override            public void run() {                System.out.println("听说我被异步调用了");            }                    });    }}
```

> 友情提示：这里说**进程内**的队列或者线程池，相对**不可靠**的原因是，队列和线程池中的任务仅仅存储在内存中，如果 JVM 进程被异常关闭，将会导致丢失，未被执行。
>
> 而分布式消息队列，异步调用会以一个消息的形式，存储在消息队列的服务器上，所以即使 JVM 进程被异常关闭，消息依然在消息队列的服务器上。
>
> 所以，使用**进程内**的队列或者线程池来实现异步调用的话，一定要尽可能的保证 JVM 进程的优雅关闭，保证它们在关闭前被执行完成。

在 [Spring Framework](https://github.com/spring-projects/spring-framework) 的 [Spring Task](https://github.com/spring-projects/spring-framework/tree/master/spring-core/src/main/java/org/springframework/core/task) 模块，提供了 [`@Async`](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/scheduling/annotation/Async.java) 注解，可以添加在方法上，自动实现该方法的异步调用。

😈 简单来说，我们可以像使用 `@Transactional` 声明式**事务**，使用 Spring Task 提供的 `@Async` 注解，😈 声明式**异步**。而在实现原理上，也是基于 Spring AOP 拦截，实现异步提交该操作到线程池中，达到异步调用的目的。

> 如果胖友看过艿艿写的 [《芋道 Spring Boot 定时任务入门》](http://www.iocoder.cn/Spring-Boot/Job/?self) 文章，就会发现 Spring Task 模块，还提供了定时任务的功能。

下面，让我们一起遨游 Spring 异步任务的海洋。

# 2. 快速入门

> 示例代码对应仓库：[lab-29-async-demo](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo) 。

本小节，我们会编写示例，对比同步调用和异步调用的性能差别，并演示 Spring `@Async` 注解的使用方式。

## 2.1 引入依赖

在 [`pom.xml`](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/pom.xml) 文件中，引入相关依赖。

```
<?xml version="1.0" encoding="UTF-8"?><project xmlns="http://maven.apache.org/POM/4.0.0"         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">    <parent>        <groupId>org.springframework.boot</groupId>        <artifactId>spring-boot-starter-parent</artifactId>        <version>2.2.1.RELEASE</version>        <relativePath/> <!-- lookup parent from repository -->    </parent>    <modelVersion>4.0.0</modelVersion>    <artifactId>lab-29-async-demo</artifactId>    <dependencies>        <!-- 引入 Spring Boot 依赖 -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter</artifactId>        </dependency>        <!-- 方便等会写单元测试 -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter-test</artifactId>            <scope>test</scope>        </dependency>    </dependencies></project>
```

因为 Spring Task 是 Spring Framework 的模块，所以在我们引入 `spring-boot-web` 依赖后，无需特别引入它。

## 2.2 Application

创建 [`Application.java`](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/Application.java) 类，配置 `@SpringBootApplication` 注解。代码如下：

```
@SpringBootApplication@EnableAsync // 开启 @Async 的支持public class Application {    public static void main(String[] args) {        SpringApplication.run(Application.class, args);    }}
```

- 在类上添加 [`@EnableAsync`](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/scheduling/annotation/EnableAsync.java) 注解，启用异步功能。

## 2.3 DemoService

在 [`cn.iocoder.springboot.lab29.asynctask.service`](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/service) 包路径下，创建 [DemoService](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/service/DemoService.java) 类。代码如下：

```
// DemoService.java@Servicepublic class DemoService {    private Logger logger = LoggerFactory.getLogger(getClass());        public Integer execute01() {        logger.info("[execute01]");        sleep(10);        return 1;    }    public Integer execute02() {        logger.info("[execute02]");        sleep(5);        return 2;    }    private static void sleep(int seconds) {        try {            Thread.sleep(seconds * 1000);        } catch (InterruptedException e) {            throw new RuntimeException(e);        }    }    }
```

- 定义了 `#execute01()` 和 `#execute02()` 方法，分别 sleep 10 秒和 5 秒，模拟耗时操作。
- 同时在每个方法里，使用 `logger` 打印日志，方便我们看到每个方法的开始执行时间，和执行所在**线程**。

## 2.4 同步调用测试

创建 [DemoServiceTest](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/test/java/cn/iocoder/springboot/lab29/asynctask/service/DemoServiceTest.java) 测试类，编写 `#task01()` 方法，同步调用 DemoService 的上述两个方法。代码如下：

```
// DemoServiceTest.java@RunWith(SpringRunner.class)@SpringBootTest(classes = Application.class)public class DemoServiceTest {    private Logger logger = LoggerFactory.getLogger(getClass());    @Autowired    private DemoService demoService;    @Test    public void task01() {        long now = System.currentTimeMillis();        logger.info("[task01][开始执行]");        demoService.execute01();        demoService.execute02();        logger.info("[task01][结束执行，消耗时长 {} 毫秒]", System.currentTimeMillis() - now);    }}
```

运行单元测试，执行日志如下：

```
2019-11-30 14:03:35.820  INFO 64639 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task01][开始执行]2019-11-30 14:03:35.828  INFO 64639 --- [           main] c.i.s.l.asynctask.service.DemoService    : [execute01]2019-11-30 14:03:45.833  INFO 64639 --- [           main] c.i.s.l.asynctask.service.DemoService    : [execute02]2019-11-30 14:03:50.834  INFO 64639 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task01][结束执行，消耗时长 15014 毫秒]
```

- DemoService 的两个方法，顺序执行，一共消耗 15 秒左右。
- DemoService 的两个方法，都在**主线程**中执行。

## 2.5 异步调用测试

修改 DemoService 的代码，增加 `#execute01()` 和 `#execute02()` 的异步调用。代码如下：

```
// DemoService.java@Asyncpublic Integer execute01Async() {    return this.execute01();}@Asyncpublic Integer execute02Async() {    return this.execute02();}
```

- 额外增加了 `#execute01Async()` 和 `#execute02Async()` 方法，主要是不想破坏上面的「2.4 同步调用测试」哈。实际上，可以在 `#execute01()` 和 `#execute02()` 方法上，添加 `@Async` 注解，实现异步调用。

修改 [DemoServiceTest](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/test/java/cn/iocoder/springboot/lab29/asynctask/service/DemoServiceTest.java) 测试类，编写 `#task02()` 方法，异步调用上述的两个方法。代码如下：

```
// DemoServiceTest.java@Testpublic void task02() {    long now = System.currentTimeMillis();    logger.info("[task02][开始执行]");    demoService.execute01Async();    demoService.execute02Async();    logger.info("[task02][结束执行，消耗时长 {} 毫秒]", System.currentTimeMillis() - now);}
```

运行单元测试，执行日志如下：

```
2019-11-30 15:57:45.809  INFO 69165 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task02][开始执行]2019-11-30 15:57:45.836  INFO 69165 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task02][结束执行，消耗时长 27 毫秒]2019-11-30 15:57:45.844  INFO 69165 --- [         task-1] c.i.s.l.asynctask.service.DemoService    : [execute01]2019-11-30 15:57:45.844  INFO 69165 --- [         task-2] c.i.s.l.asynctask.service.DemoService    : [execute02]
```

- DemoService 的两个方法，异步执行，所以主线程只消耗 27 毫秒左右。**注意**，实际这两个方法，并没有执行完成。
- DemoService 的两个方法，都在**异步的线程池**中，进行执行。

## 2.6 等待异步调用完成测试

在 [「2.5 异步调用测试」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 中，两个方法只是发布异步调用，并未执行完成。在一些业务场景中，我们希望达到异步调用的效果，同时主线程**阻塞等待**异步调用的结果。

修改 DemoService 的代码，增加 `#execute01()` 和 `#execute02()` 的异步调用，**并返回 Future 对象**。代码如下：

```
// DemoService.java@Asyncpublic Future<Integer> execute01AsyncWithFuture() {    return AsyncResult.forValue(this.execute01());}@Asyncpublic Future<Integer> execute02AsyncWithFuture() {    return AsyncResult.forValue(this.execute02());}
```

- 相比 [「2.5 异步调用测试」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 的两个方法，我们额外增加调用 `AsyncResult#forValue(V value)` 方法，返回带有执行结果的 Future 对象。

修改 [DemoServiceTest](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/test/java/cn/iocoder/springboot/lab29/asynctask/service/DemoServiceTest.java) 测试类，编写 `#task03()` 方法，异步调用上述的两个方法，**并阻塞等待执行完成**。代码如下：

```
// DemoServiceTest.java@Testpublic void task03() throws ExecutionException, InterruptedException {    long now = System.currentTimeMillis();    logger.info("[task03][开始执行]");    // <1> 执行任务    Future<Integer> execute01Result = demoService.execute01AsyncWithFuture();    Future<Integer> execute02Result = demoService.execute02AsyncWithFuture();    // <2> 阻塞等待结果    execute01Result.get();    execute02Result.get();    logger.info("[task03][结束执行，消耗时长 {} 毫秒]", System.currentTimeMillis() - now);}
```

- `<1>` 处，异步调用两个方法，并返回对应的 Future 对象。这样，这两个异步调用的逻辑，可以**并行**执行。
- `<2>` 处，分别调用两个 Future 对象的 `#get()` 方法，阻塞等待结果。

运行单元测试，执行日志如下：

```
2019-11-30 16:10:22.226  INFO 69641 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task03][开始执行]2019-11-30 16:10:22.272  INFO 69641 --- [         task-1] c.i.s.l.asynctask.service.DemoService    : [execute01]2019-11-30 16:10:22.272  INFO 69641 --- [         task-2] c.i.s.l.asynctask.service.DemoService    : [execute02]2019-11-30 16:10:32.276  INFO 69641 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task03][结束执行，消耗时长 10050 毫秒]
```

- DemoService 的两个方法，异步执行，因为主线程阻塞等待执行结果，所以消耗 10 秒左右。**当同时有多个异步调用，并阻塞等待执行结果，消耗时长由最慢的异步调用的逻辑所决定。**
- DemoService 的两个方法，都在**异步的线程池**中，进行执行。

> 下面[「2.7 应用配置文件」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#)小节，是补充知识，建议看看。

## 2.7 应用配置文件

在 [`application.yml`](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/main/resources/application.yaml) 中，添加 Spring Task 定时任务的配置，如下：

```
spring:  task:    # Spring 执行器配置，对应 TaskExecutionProperties 配置类。对于 Spring 异步任务，会使用该执行器。    execution:      thread-name-prefix: task- # 线程池的线程名的前缀。默认为 task- ，建议根据自己应用来设置      pool: # 线程池相关        core-size: 8 # 核心线程数，线程池创建时候初始化的线程数。默认为 8 。        max-size: 20 # 最大线程数，线程池最大的线程数，只有在缓冲队列满了之后，才会申请超过核心线程数的线程。默认为 Integer.MAX_VALUE        keep-alive: 60s # 允许线程的空闲时间，当超过了核心线程之外的线程，在空闲时间到达之后会被销毁。默认为 60 秒        queue-capacity: 200 # 缓冲队列大小，用来缓冲执行任务的队列的大小。默认为 Integer.MAX_VALUE 。        allow-core-thread-timeout: true # 是否允许核心线程超时，即开启线程池的动态增长和缩小。默认为 true 。      shutdown:        await-termination: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true        await-termination-period: 60 # 等待任务完成的最大时长，单位为秒。默认为 0 ，根据自己应用来设置
```

- 在 `spring.task.execution` 配置项，Spring Task 调度任务的配置，对应 [TaskExecutionProperties](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskExecutionProperties.java) 配置类。
- Spring Boot [TaskExecutionAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskExecutionAutoConfiguration.java) 自动化配置类，实现 Spring Task 的自动配置，创建 [ThreadPoolTaskExecutor](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/scheduling/concurrent/ThreadPoolTaskExecutor.java) 基于线程池的任务执行器。本质上，ThreadPoolTaskExecutor 是基于 ThreadPoolExecutor 的封装，主要增加提交任务，返回 **ListenableFuture** 对象的功能。

**注意**，`spring.task.execution.shutdown` 配置项，是为了实现 Spring Task 异步任务的优雅关闭。我们想象一下，如果异步任务在执行的过程中，如果应用开始关闭，把异步任务需要使用到的 Spring Bean 进行销毁，例如说数据库连接池，那么此时异步任务还在执行中，一旦需要访问数据库，可能会导致报错。

- 所以，通过配置 `await-termination = true` ，实现应用关闭时，等待异步任务执行完成。这样，应用在关闭的时，Spring 会优先等待 ThreadPoolTaskScheduler 执行完任务之后，再开始 Spring Bean 的销毁。
- 同时，又考虑到我们不可能无限等待异步任务全部执行结束，因此可以配置 `await-termination-period = 60` ，等待任务完成的最大时长，单位为秒。具体设置多少的等待时长，可以根据自己应用的需要。

# 3. 异步回调

> 示例代码对应仓库：[lab-29-async-demo](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo) 。

😈 异步 + 回调，快活似神仙。所以本小节我们来看看，如何在异步调用完成后，实现**自定义回调**。

考虑到让胖友更加理解 Spring Task 异步回调是如何实现的，我们会在 [「3.1 AsyncResult」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 和 [「3.2 ListenableFutureTask」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#)小节进行部分源码解析，请保持淡定。如果不想看的胖友，可以直接看 [「3.3 具体示例」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 小节。

> 友情提示：该示例，基于 [「2. 快速入门」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 的 [lab-29-async-demo](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo) 的基础上，继续改造。

## 3.1 AsyncResult

在 [「2.6 等待异步调用完成测试」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 中，我们看到了 [AsyncResult](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/scheduling/annotation/AsyncResult.java) 类，表示异步结果。返回结果分成两种情况：

- 执行成功时，调用

   

  ```
  AsyncResult#forValue(V value)
  ```

   

  静态方法，返回成功的 ListenableFuture 对象。代码如下：

  ```
  // AsyncResult.java@Nullableprivate final V value;public static <V> ListenableFuture<V> forValue(V value) {	return new AsyncResult<>(value, null);}
  ```

- 执行异常时，调用

   

  ```
  AsyncResult#forExecutionException(Throwable ex)
  ```

   

  静态方法，返回异常的 ListenableFuture 对象。代码如下：

  ```
  // AsyncResult.java@Nullableprivate final Throwable executionException;public static <V> ListenableFuture<V> forExecutionException(Throwable ex) {	return new AsyncResult<>(null, ex);}
  ```

同时，AsyncResult 实现了 [ListenableFuture](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/concurrent/ListenableFuture.java) 接口，提供异步执行结果的回调处理。这里，我们先来看看 ListenableFuture 接口。代码如下：

```
// ListenableFuture.javapublic interface ListenableFuture<T> extends Future<T> {    // 添加回调方法，统一处理成功和异常的情况。    void addCallback(ListenableFutureCallback<? super T> callback);        // 添加成功和失败的回调方法，分别处理成功和异常的情况。    void addCallback(SuccessCallback<? super T> successCallback, FailureCallback failureCallback);        // 将 ListenableFuture 转换成 JDK8 提供的 CompletableFuture 。    // 这样，后续我们可以使用 ListenableFuture 来设置回调    // 不了解 CompletableFuture 的胖友，可以看看 https://colobu.com/2016/02/29/Java-CompletableFuture/ 文章。    default CompletableFuture<T> completable() {    	CompletableFuture<T> completable = new DelegatingCompletableFuture<>(this);    	addCallback(completable::complete, completable::completeExceptionally);    	return completable;    }    }
```

- 看下每个接口方法上的注释。

因为 ListenableFuture 继承了 [Future](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/java/util/concurrent/Future.java) 接口，所以 AsyncResult 也需要实现 Future 接口。这里，我们再来看看 Future 接口。代码如下：

```
// Future.javapublic interface Future<V> {    // 获取异步执行的结果，如果没有结果可用，此方法会阻塞直到异步计算完成。    V get() throws InterruptedException, ExecutionException;    // 获取异步执行结果，如果没有结果可用，此方法会阻塞，但是会有时间限制，如果阻塞时间超过设定的 timeout 时间，该方法将抛出异常。    V get(long timeout, TimeUnit unit)        throws InterruptedException, ExecutionException, TimeoutException;    // 如果任务执行结束，无论是正常结束或是中途取消还是发生异常，都返回 true 。    boolean isDone();    // 如果任务完成前被取消，则返回 true 。    boolean isCancelled();    // 如果任务还没开始，执行 cancel(...) 方法将返回 false；    // 如果任务已经启动，执行 cancel(true) 方法将以中断执行此任务线程的方式来试图停止任务，如果停止成功，返回 true ；    // 当任务已经启动，执行c ancel(false) 方法将不会对正在执行的任务线程产生影响(让线程正常执行到完成)，此时返回 false ；    // 当任务已经完成，执行 cancel(...) 方法将返回 false 。    // mayInterruptRunning 参数表示是否中断执行中的线程。    boolean cancel(boolean mayInterruptIfRunning);}
```

- 如上注释内容，参考自 [《Java 多线程编程：Callable、Future 和 FutureTask 浅析》](https://blog.csdn.net/javazejian/article/details/50896505) 文章。

AsyncResult 对 ListenableFuture 定义的 `#addCallback(...)` 接口方法，实现代码如下：

```
// AsyncResult.java@Overridepublic void addCallback(ListenableFutureCallback<? super V> callback) {	addCallback(callback, callback);}@Overridepublic void addCallback(SuccessCallback<? super V> successCallback, FailureCallback failureCallback) {	try {		if (this.executionException != null) { // <1>			failureCallback.onFailure(exposedException(this.executionException));		} else { // <2>			successCallback.onSuccess(this.value);		}	} catch (Throwable ex) { // <3>		// Ignore	}}// 从 ExecutionException 中，获得原始异常。private static Throwable exposedException(Throwable original) {	if (original instanceof ExecutionException) {		Throwable cause = original.getCause();		if (cause != null) {			return cause;		}	}	return original;}
```

- [ListenableFutureCallback](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/concurrent/ListenableFutureCallback.java) 接口，同时继承 [SuccessCallback](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/concurrent/SuccessCallback.java) 和 [FailureCallback](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/concurrent/FailureCallback.java) 接口。
- `<1>` 处，如果是异常的结果，调用 FailureCallback 的回调。
- `<2>` 处，如果是正常的结果，调用 SuccessCallback 的回调。
- `<3>` 处，如果回调的逻辑发生异常，直接忽略。😈 所有，如果如果有多个回调，如果有一个回调发生异常，不会影响后续的回调。

(⊙o⊙)… 不过有点懵逼的是，不是应该在异步调用执行成功后，才进行回调么？！怎么这里一添加回调方法，就直接执行了？！不要着急，答案在 [「3.2 ListenableFutureTask」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 中解答。

实际上，AsyncResult 是作为异步执行的**结果**。既然是结果，执行就已经完成。所以，在我们调用 `#addCallback(...)` 接口方法来添加回调时，**必然直接使用回调处理执行的结果**。

AsyncResult 对 ListenableFuture 定义的 `#completable(...)` 接口方法，实现代码如下：

```
// AsyncResult.java@Overridepublic CompletableFuture<V> completable() {	if (this.executionException != null) {		CompletableFuture<V> completable = new CompletableFuture<>();		completable.completeExceptionally(exposedException(this.executionException));		return completable;	} else {		return CompletableFuture.completedFuture(this.value);	}}
```

- 直接将结果包装成 CompletableFuture 对象。

AsyncResult 对 Future 定义的所有方法，实现代码如下：

```
// AsyncResult.java@Overridepublic boolean cancel(boolean mayInterruptIfRunning) {	return false; // 因为是 AsyncResult 是执行结果，所以直接返回 false 表示取消失败。}@Overridepublic boolean isCancelled() {	return false; // 因为是 AsyncResult 是执行结果，所以直接返回 false 表示未取消。}@Overridepublic boolean isDone() {	return true; // 因为是 AsyncResult 是执行结果，所以直接返回 true 表示已完成。}@Override@Nullablepublic V get() throws ExecutionException {	// 如果发生异常，则抛出该异常。	if (this.executionException != null) {		throw (this.executionException instanceof ExecutionException ?				(ExecutionException) this.executionException :				new ExecutionException(this.executionException));	}	// 如果执行成功，则返回该 value 结果	return this.value;}@Override@Nullablepublic V get(long timeout, TimeUnit unit) throws ExecutionException {	return get();}
```

- 胖友自己看看代码上的注释。

😈 看到这里，相信很多胖友会是一脸懵逼，淡定淡定。看源码这个事儿，总是柳暗花明又一村。

## 3.2 ListenableFutureTask

在我们调用使用 `@Async` 注解的方法时，如果方法返回的类型是 ListenableFuture 的情况下，实际方法返回的是 [ListenableFutureTask](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/concurrent/ListenableFutureTask.java) 对象。

> 感兴趣的胖友，可以看看 [AsyncExecutionInterceptor](https://github.com/spring-projects/spring-framework/blob/master/spring-aop/src/main/java/org/springframework/aop/interceptor/AsyncExecutionInterceptor.java) 类、[《Spring 异步调用原理及Spring AOP 拦截器链原理》](https://zhuanlan.zhihu.com/p/59633595) 文章。

ListenableFutureTask 类，**也实现 ListenableFuture 接口**，继承 [FutureTask](https://github.com/frohoff/jdk8u-jdk/blob/master/src/share/classes/java/util/concurrent/FutureTask.java) 类，ListenableFuture 的 FutureTask 实现类。

ListenableFutureTask 对 ListenableFuture 定义的 `#addCallback(...)` 方法，实现代码如下：

```
// ListenableFutureTask.javaprivate final ListenableFutureCallbackRegistry<T> callbacks = new ListenableFutureCallbackRegistry<T>();@Overridepublic void addCallback(ListenableFutureCallback<? super T> callback) {	this.callbacks.addCallback(callback);}@Overridepublic void addCallback(SuccessCallback<? super T> successCallback, FailureCallback failureCallback) {	this.callbacks.addSuccessCallback(successCallback);	this.callbacks.addFailureCallback(failureCallback);}
```

- 暂存回调到 [ListenableFutureCallbackRegistry](https://github.com/spring-projects/spring-framework/blob/master/spring-core/src/main/java/org/springframework/util/concurrent/ListenableFutureCallbackRegistry.java) 中先。😈 这样看起来，和我们想象中的异步回调有点像了。

ListenableFutureTask 对 FutureTask 已实现的 `#done()` 方法，进行**重写**。实现代码如下：

```
// ListenableFutureTask.java@Overrideprotected void done() {	Throwable cause;	try {	   // <1> 获得执行结果		T result = get();		// <2.1> 执行成功，执行成功的回调		this.callbacks.success(result);		return;	} catch (InterruptedException ex) { // 如果有中断异常 InterruptedException ，则打断当前线程，并直接返回		Thread.currentThread().interrupt();		return;	} catch (ExecutionException ex) { // 如果有 ExecutionException 异常，获得其真实的异常，并设置到 cause 中		cause = ex.getCause();		if (cause == null) {			cause = ex;		}	} catch (Throwable ex) { // 设置异常到 cause 中		cause = ex;	}	// 执行异常，执行异常的回调	this.callbacks.failure(cause);}
```

- `<1>` 处，调用 `#get()` 方法，获得执行结果。
- `<2.1>` 处，执行成功，执行成功的回调。
- `<2.2>` 处，执行异常，执行异常的回调。

这样一看，是不是对 AsyncResult 和 ListenableFutureTask 就有点感觉了。

## 3.3 具体示例

下面，让我们来写一个异步回调的示例。修改 DemoService 的代码，增加 `#execute02()` 的异步调用，**并返回 ListenableFuture 对象**。代码如下：

```
// DemoService.java@Asyncpublic ListenableFuture<Integer> execute01AsyncWithListenableFuture() {    try {        return AsyncResult.forValue(this.execute02());    } catch (Throwable ex) {        return AsyncResult.forExecutionException(ex);    }}
```

- 根据执行的结果，包装出成功还是异常的 AsyncResult 对象。

修改 [DemoServiceTest](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/test/java/cn/iocoder/springboot/lab29/asynctask/service/DemoServiceTest.java) 测试类，编写 `#task04()` 方法，异步调用上述的方法，在塞等待执行完成的同时，添加相应的回调 Callback 方法。代码如下：

```
// DemoServiceTest.java@Testpublic void task04() throws ExecutionException, InterruptedException {    long now = System.currentTimeMillis();    logger.info("[task04][开始执行]");    // <1> 执行任务    ListenableFuture<Integer> execute01Result = demoService.execute01AsyncWithListenableFuture();    logger.info("[task04][execute01Result 的类型是：({})]",execute01Result.getClass().getSimpleName());    execute01Result.addCallback(new SuccessCallback<Integer>() { // <2.1> 增加成功的回调        @Override        public void onSuccess(Integer result) {            logger.info("[onSuccess][result: {}]", result);        }    }, new FailureCallback() { // <2.1> 增加失败的回调        @Override        public void onFailure(Throwable ex) {            logger.info("[onFailure][发生异常]", ex);        }    });    execute01Result.addCallback(new ListenableFutureCallback<Integer>() { // <2.2> 增加成功和失败的统一回调        @Override        public void onSuccess(Integer result) {            logger.info("[onSuccess][result: {}]", result);        }        @Override        public void onFailure(Throwable ex) {            logger.info("[onFailure][发生异常]", ex);        }    });    // <3> 阻塞等待结果    execute01Result.get();    logger.info("[task04][结束执行，消耗时长 {} 毫秒]", System.currentTimeMillis() - now);}
```

- ```
  <1>
  ```

   

  处，调用

   

  ```
  DemoService#execute01AsyncWithListenableFuture()
  ```

   

  方法，异步调用该方法，并返回 ListenableFutureTask 对象。这里，我们看下打印的日志。

  ```
  2019-11-30 19:17:51.320  INFO 77624 --- [           main] c.i.s.l.a.service.DemoServiceTest        : [task04][execute01Result 的类型是：(ListenableFutureTask)]
  ```

- `<2.1>` 处，增加成功的回调和失败的回调。

- `<2.2>` 处，增加成功和失败的统一回调。

- ```
  <3>
  ```

   

  处，阻塞等待结果。执行完成后，我们会看到回调被执行，打印日志如下：

  ```
  2019-11-30 19:17:56.330  INFO 77624 --- [         task-1] c.i.s.l.a.service.DemoServiceTest        : [onSuccess][result: 2]2019-11-30 19:17:56.331  INFO 77624 --- [         task-1] c.i.s.l.a.service.DemoServiceTest        : [onSuccess][result: 2]
  ```

# 4. 异步异常处理器

> 示例代码对应仓库：[lab-29-async-demo](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo) 。

在 [《芋道 Spring Boot SpringMVC 入门》](http://www.iocoder.cn/Spring-Boot/SpringMVC/?self) 的 [「5. 全局异常处理」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 中，我们实现了对 SpringMVC 请求异常的全局处理。那么，Spring Task 异步调用异常是否有全局处理呢？答案是有，通过实现 [AsyncUncaughtExceptionHandler](https://github.com/spring-projects/spring-framework/blob/master/spring-aop/src/main/java/org/springframework/aop/interceptor/AsyncUncaughtExceptionHandler.java) 接口，达到对异步调用的异常的统一处理。

> 友情提示：该示例，基于 [「2. 快速入门」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 的 [lab-29-async-demo](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo) 的基础上，继续改造。

## 4.1 GlobalAsyncExceptionHandler

在 [`cn.iocoder.springboot.lab29.asynctask.core.async`](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/core/async) 包路径，创建 [GlobalAsyncExceptionHandler](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/core/async/GlobalAsyncExceptionHandler.java) 类，全局统一的异步调用异常的处理器。代码如下：

```
// GlobalAsyncExceptionHandler.java@Componentpublic class GlobalAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {    private Logger logger = LoggerFactory.getLogger(getClass());    @Override    public void handleUncaughtException(Throwable ex, Method method, Object... params) {        logger.error("[handleUncaughtException][method({}) params({}) 发生异常]",                method, params, ex);    }}
```

- 类上，我们添加了 `@Component` 注解，考虑到胖友可能会注入一些 Spring Bean 到属性中。
- 实现 `#handleUncaughtException(Throwable ex, Method method, Object... params)` 方法，打印异常日志。😈 这样，后续如果我们接入 ELK ，就可以基于该异常日志进行告警。

**注意**，AsyncUncaughtExceptionHandler 只能拦截**返回类型非 Future** 的异步调用方法。通过看 [`AsyncExecutionAspectSupport#handleError(Throwable ex, Method method, Object... params)`](https://github.com/spring-projects/spring-framework/blob/master/spring-aop/src/main/java/org/springframework/aop/interceptor/AsyncExecutionAspectSupport.java#L295-L321) 的源码，可以很容易得到这个结论，代码如下：

```
// AsyncExecutionAspectSupport.javaprotected void handleError(Throwable ex, Method method, Object... params) throws Exception {	// 重点！！！如果返回类型是 Future ，则直接抛出该异常。	if (Future.class.isAssignableFrom(method.getReturnType())) {		ReflectionUtils.rethrowException(ex);	} else {		// 否则，交给 AsyncUncaughtExceptionHandler 来处理。		// Could not transmit the exception to the caller with default executor		try {			this.exceptionHandler.obtain().handleUncaughtException(ex, method, params);		} catch (Throwable ex2) {			logger.warn("Exception handler for async method '" + method.toGenericString() +					"' threw unexpected exception itself", ex2);		}	}}
```

- 对了，AsyncExecutionAspectSupport 是 AsyncExecutionInterceptor 的父类哟。

所以哟，返回类型为 Future 的异步调用方法，需要通过[「3. 异步回调」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#)来处理。

## 4.2 AsyncConfig

在 [`cn.iocoder.springboot.lab29.asynctask.config`](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/config) 包路径，创建 [AsyncConfig](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/main/java/cn/iocoder/springboot/lab29/asynctask/config/AsyncConfig.java) 类，配置异常处理器。代码如下：

```
// AsyncConfig.java@Configuration@EnableAsync // 开启 @Async 的支持public class AsyncConfig implements AsyncConfigurer {    @Autowired    private GlobalAsyncExceptionHandler exceptionHandler;    @Override    public Executor getAsyncExecutor() {        return null;    }    @Override    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {        return exceptionHandler;    }}
```

- 在类上添加 [`@EnableAsync`](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/scheduling/annotation/EnableAsync.java) 注解，启用异步功能。这样[「2. Application」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 的 `@EnableAsync` 注解，也就可以去掉了。
- 实现 [AsyncConfigurer](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/scheduling/annotation/AsyncConfigurer.java) 接口，实现异步相关的全局配置。😈 此时此刻，胖友有没想到 SpringMVC 的 [WebMvcConfigurer](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/java/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.java) 接口。
- 实现 `#getAsyncUncaughtExceptionHandler()` 方法，返回我们定义的 GlobalAsyncExceptionHandler 对象。
- 实现 `#getAsyncExecutor()` 方法，返回 Spring Task 异步任务的**默认执行器**。这里，我们返回了 `null` ，并未定义默认执行器。所以最终会使用 [TaskExecutionAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskExecutionAutoConfiguration.java) 自动化配置类创建出来的 ThreadPoolTaskExecutor 任务执行器，作为默认执行器。

## 4.3 DemoService

修改 DemoService 的代码，增加 `#zhaoDaoNvPengYou(...)` 的异步调用。代码如下：

```
// DemoService.java@Asyncpublic Integer zhaoDaoNvPengYou(Integer a, Integer b) {    throw new RuntimeException("程序员不需要女朋友");}
```

- 直接给想要找女朋友的程序员，抛出该异常。

## 4.4 简单测试

修改 [DemoServiceTest](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-demo/src/test/java/cn/iocoder/springboot/lab29/asynctask/service/DemoServiceTest.java) 测试类，编写 `#testZhaoDaoNvPengYou()` 方法，异步调用上述的方法。代码如下：

```
// DemoServiceTest.java@Testpublic void testZhaoDaoNvPengYou() throws InterruptedException {    demoService.zhaoDaoNvPengYou(1, 2);    // sleep 1 秒，保证异步调用的执行    Thread.sleep(1000);}
```

运行单元测试，执行日志如下：

```
2019-11-30 09:22:52.962 ERROR 86590 --- [         task-1] .i.s.l.a.c.a.GlobalAsyncExceptionHandler : [handleUncaughtException][method(public java.lang.Integer cn.iocoder.springboot.lab29.asynctask.service.DemoService.zhaoDaoNvPengYou(java.lang.Integer,java.lang.Integer)) params([1, 2]) 发生异常]java.lang.RuntimeException: 程序员不需要女朋友
```

- 😈 异步调用的异常成功被 GlobalAsyncExceptionHandler 拦截。

# 5. 自定义执行器

> 示例代码对应仓库：[lab-29-async-two](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-two) 。

在 [「2. 快速入门」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 中，我们使用 Spring Boot [TaskExecutionAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskExecutionAutoConfiguration.java) 自动化配置类，实现自动配置 ThreadPoolTaskExecutor 任务执行器。

本小节，我们希望**两个**自定义 ThreadPoolTaskExecutor 任务执行器，实现不同方法，分别使用这两个 ThreadPoolTaskExecutor 任务执行器。

> 友情提示：考虑到不破坏上面入门的示例，所以我们新建了 [lab-29-async-two](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-two) 项目。

## 5.1 引入依赖

在 [`pom.xml`](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-two/pom.xml) 文件中，引入相关依赖。

```
<?xml version="1.0" encoding="UTF-8"?><project xmlns="http://maven.apache.org/POM/4.0.0"         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">    <parent>        <groupId>org.springframework.boot</groupId>        <artifactId>spring-boot-starter-parent</artifactId>        <version>2.2.1.RELEASE</version>        <relativePath/> <!-- lookup parent from repository -->    </parent>    <modelVersion>4.0.0</modelVersion>    <artifactId>lab-29-async-demo</artifactId>    <dependencies>        <!-- 引入 Spring Boot 依赖 -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter</artifactId>        </dependency>        <!-- 方便等会写单元测试 -->        <dependency>            <groupId>org.springframework.boot</groupId>            <artifactId>spring-boot-starter-test</artifactId>            <scope>test</scope>        </dependency>    </dependencies></project>
```

- 和 [「2.1 引入依赖」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#) 一致。

## 5.2 应用配置文件

在 [`application.yml`](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-two/src/main/resources/application.yaml) 中，添加 Spring Task 定时任务的配置，如下：

```
spring:  task:    # Spring 执行器配置，对应 TaskExecutionProperties 配置类。对于 Spring 异步任务，会使用该执行器。    execution-one:      thread-name-prefix: task-one- # 线程池的线程名的前缀。默认为 task- ，建议根据自己应用来设置      pool: # 线程池相关        core-size: 8 # 核心线程数，线程池创建时候初始化的线程数。默认为 8 。        max-size: 20 # 最大线程数，线程池最大的线程数，只有在缓冲队列满了之后，才会申请超过核心线程数的线程。默认为 Integer.MAX_VALUE        keep-alive: 60s # 允许线程的空闲时间，当超过了核心线程之外的线程，在空闲时间到达之后会被销毁。默认为 60 秒        queue-capacity: 200 # 缓冲队列大小，用来缓冲执行任务的队列的大小。默认为 Integer.MAX_VALUE 。        allow-core-thread-timeout: true # 是否允许核心线程超时，即开启线程池的动态增长和缩小。默认为 true 。      shutdown:        await-termination: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true        await-termination-period: 60 # 等待任务完成的最大时长，单位为秒。默认为 0 ，根据自己应用来设置    # Spring 执行器配置，对应 TaskExecutionProperties 配置类。对于 Spring 异步任务，会使用该执行器。    execution-two:      thread-name-prefix: task-two- # 线程池的线程名的前缀。默认为 task- ，建议根据自己应用来设置      pool: # 线程池相关        core-size: 8 # 核心线程数，线程池创建时候初始化的线程数。默认为 8 。        max-size: 20 # 最大线程数，线程池最大的线程数，只有在缓冲队列满了之后，才会申请超过核心线程数的线程。默认为 Integer.MAX_VALUE        keep-alive: 60s # 允许线程的空闲时间，当超过了核心线程之外的线程，在空闲时间到达之后会被销毁。默认为 60 秒        queue-capacity: 200 # 缓冲队列大小，用来缓冲执行任务的队列的大小。默认为 Integer.MAX_VALUE 。        allow-core-thread-timeout: true # 是否允许核心线程超时，即开启线程池的动态增长和缩小。默认为 true 。      shutdown:        await-termination: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true        await-termination-period: 60 # 等待任务完成的最大时长，单位为秒。默认为 0 ，根据自己应用来设置
```

- 在 `spring.task` 配置项下，我们新增了 `execution-one` 和 `execution-two` 两个执行器的配置。在格式上，我们保持和在[「2.7 应用配置文件」](http://www.iocoder.cn/Spring-Boot/Async-Job/?github#)看到的 `spring.task.exeuction` 一致，方便我们后续复用 TaskExecutionProperties 属性配置类来映射。

## 5.3 AsyncConfig

在 [`cn.iocoder.springboot.lab29.asynctask.config`](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-two/src/main/java/cn/iocoder/springboot/lab29/asynctask/config) 包路径，创建 [AsyncConfig](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-two/src/main/java/cn/iocoder/springboot/lab29/asynctask/config/AsyncConfig.java) 类，配置两个执行器。代码如下：

```
// AsyncConfig.java@Configuration@EnableAsync // 开启 @Async 的支持public class AsyncConfig {    public static final String EXECUTOR_ONE_BEAN_NAME = "executor-one";    public static final String EXECUTOR_TWO_BEAN_NAME = "executor-two";    @Configuration    public static class ExecutorOneConfiguration {        @Bean(name = EXECUTOR_ONE_BEAN_NAME + "-properties")        @Primary        @ConfigurationProperties(prefix = "spring.task.execution-one") // 读取 spring.task.execution-one 配置到 TaskExecutionProperties 对象        public TaskExecutionProperties taskExecutionProperties() {            return new TaskExecutionProperties();        }        @Bean(name = EXECUTOR_ONE_BEAN_NAME)        public ThreadPoolTaskExecutor threadPoolTaskExecutor() {            // 创建 TaskExecutorBuilder 对象            TaskExecutorBuilder builder = createTskExecutorBuilder(this.taskExecutionProperties());            // 创建 ThreadPoolTaskExecutor 对象            return builder.build();        }    }    @Configuration    public static class ExecutorTwoConfiguration {        @Bean(name = EXECUTOR_TWO_BEAN_NAME + "-properties")        @ConfigurationProperties(prefix = "spring.task.execution-two") // 读取 spring.task.execution-two 配置到 TaskExecutionProperties 对象        public TaskExecutionProperties taskExecutionProperties() {            return new TaskExecutionProperties();        }        @Bean(name = EXECUTOR_TWO_BEAN_NAME)        public ThreadPoolTaskExecutor threadPoolTaskExecutor() {            // 创建 TaskExecutorBuilder 对象            TaskExecutorBuilder builder = createTskExecutorBuilder(this.taskExecutionProperties());            // 创建 ThreadPoolTaskExecutor 对象            return builder.build();        }    }    private static TaskExecutorBuilder createTskExecutorBuilder(TaskExecutionProperties properties) {        // Pool 属性        TaskExecutionProperties.Pool pool = properties.getPool();        TaskExecutorBuilder builder = new TaskExecutorBuilder();        builder = builder.queueCapacity(pool.getQueueCapacity());        builder = builder.corePoolSize(pool.getCoreSize());        builder = builder.maxPoolSize(pool.getMaxSize());        builder = builder.allowCoreThreadTimeOut(pool.isAllowCoreThreadTimeout());        builder = builder.keepAlive(pool.getKeepAlive());        // Shutdown 属性        TaskExecutionProperties.Shutdown shutdown = properties.getShutdown();        builder = builder.awaitTermination(shutdown.isAwaitTermination());        builder = builder.awaitTerminationPeriod(shutdown.getAwaitTerminationPeriod());        // 其它基本属性        builder = builder.threadNamePrefix(properties.getThreadNamePrefix());//        builder = builder.customizers(taskExecutorCustomizers.orderedStream()::iterator);//        builder = builder.taskDecorator(taskDecorator.getIfUnique());        return builder;    }}
```

- 参考 Spring Boot [TaskExecutionAutoConfiguration](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskExecutionAutoConfiguration.java) 自动化配置类，我们创建了 ExecutorOneConfiguration 和 ExecutorTwoConfiguration 配置类，来分别创建 Bean 名字为 `executor-one` 和 `executor-two` 两个执行器。

## 5.4 DemoService

在 [`cn.iocoder.springboot.lab29.asynctask.service`](https://github.com/YunaiV/SpringBoot-Labs/tree/master/lab-29/lab-29-async-two/src/main/java/cn/iocoder/springboot/lab29/asynctask/service) 包路径下，创建 [DemoService](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-two/src/main/java/cn/iocoder/springboot/lab29/asynctask/service/DemoService.java) 类。代码如下：

```
// DemoService.java@Servicepublic class DemoService {    private Logger logger = LoggerFactory.getLogger(getClass());    @Async(AsyncConfig.EXECUTOR_ONE_BEAN_NAME)    public Integer execute01() {        logger.info("[execute01]");        return 1;    }    @Async(AsyncConfig.EXECUTOR_TWO_BEAN_NAME)    public Integer execute02() {        logger.info("[execute02]");        return 2;    }}
```

- 在 `@Async` 注解上，我们设置了其使用的执行器的 Bean 名字。

## 5.5 简单测试

创建 [DemoServiceTest](https://github.com/YunaiV/SpringBoot-Labs/blob/master/lab-29/lab-29-async-two/src/test/java/cn/iocoder/springboot/lab29/asynctask/service/DemoServiceTest.java) 测试类，编写 `#testExecute()` 方法，异步调用 DemoService 的上述两个方法。代码如下：

```
// DemoServiceTest.java@RunWith(SpringRunner.class)@SpringBootTest(classes = Application.class)public class DemoServiceTest {    @Autowired    private DemoService demoService;    @Test    public void testExecute() throws InterruptedException {        demoService.execute01();        demoService.execute02();        // sleep 1 秒，保证异步调用的执行        Thread.sleep(1000);    }}
```

运行单元测试，执行日志如下：

```
2019-11-30 10:25:53.068  INFO 89290 --- [     task-one-1] c.i.s.l.asynctask.service.DemoService    : [execute01]2019-11-30 10:25:53.068  INFO 89290 --- [     task-two-1] c.i.s.l.asynctask.service.DemoService    : [execute02]
```

- 从日志中，我们可以看到，`#execute01()` 方法在 `executor-one` 执行器中执行，而 `#execute02()` 方法在 `executor-two` 执行器中执行。符合预期~

# 666. 彩蛋

😈 发现自己真是一个啰嗦的老男孩，挺简单一东西，结果又写了老长一篇。不过最后还是要唠叨下，如果胖友使用 Spring Task 的异步任务，一定要注意两个点：

- JVM 应用的正常优雅关闭，保证异步任务都被执行完成。
- 编写异步异常处理器 GlobalAsyncExceptionHandler ，记录异常日志，进行监控告警。

嗯~~~如果觉得不过瘾的胖友，可以再去看看 [《Spring Framework Documentation —— Task Execution and Scheduling》](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/integration.html#scheduling) 文档。

不过呢，Spring Task 异步任务，在项目中使用的并不多，更多的选择，还是可靠的分布式队列，嘿嘿。当然，艿艿在自己的开源项目 [onemall](https://github.com/YunaiV/onemall) 中，使用 [AccessLogInterceptor](https://github.com/YunaiV/onemall/blob/74724637b7408461e6570855172c753337293b30/common/mall-spring-boot/src/main/java/cn/iocoder/mall/spring/boot/web/interceptor/AccessLogInterceptor.java) 拦截器，记录访问日志到数据库。因为访问日志更多是用于监控和排查问题，所以即使有一定的丢失，影响也不大。