[toc]

## 本篇要点

- 介绍观察者模式和发布订阅模式的区别。
- SpringBoot快速入门事件监听。

## 什么是观察者模式？

观察者模式是经典行为型设计模式之一。

在GoF的《设计模式》中，观察者模式的定义：**在对象之间定义一个一对多的依赖，当一个对象状态改变的时候，所有依赖的对象都会自动收到通知**。如果你觉得比较抽象，接下来这个例子应该会让你有所感觉：

就拿用户注册功能为例吧，假设用户注册成功之后，我们将会发送邮件，优惠券等等操作，很容易就能写出下面的逻辑：

```java
@RestController
@RequestMapping("/user")
public class SimpleUserController {

    @Autowired
    private SimpleEmailService emailService;

    @Autowired
    private SimpleCouponService couponService;

    @Autowired
    private SimpleUserService userService;

    @GetMapping("/register")
    public String register(String username) {
        // 注册
        userService.register(username);
        // 发送邮件
        emailService.sendEmail(username);
        // 发送优惠券
        couponService.addCoupon(username);
        return "注册成功!";
    }
}
```

这样写会有什么问题呢？受王争老师启发：

- 方法调用时，同步阻塞导致响应变慢，需要异步非阻塞的解决方案。

- 注册接口此时做的事情：注册，发邮件，优惠券，违反单一职责的原则。当然，如果后续没有拓展和修改的需求，这样子倒可以接受。
- 如果后续注册的需求频繁变更，相应就需要频繁变更register方法，违反了开闭原则。

针对以上的问题，我们想一想解决的方案：

一、异步非阻塞的效果可以新开一个线程执行耗时的发送邮件任务，但频繁地创建和销毁线程比较耗时，并且并发线程数无法控制，创建过多的线程会导致堆栈溢出。

二、使用线程池执行任务解决上述问题。

```java
@Service
@Slf4j
public class SimpleEmailService {
	// 启动一个线程执行耗时操作
    public void sendEmail(String username) {
        Thread thread = new Thread(()->{
            try {
                // 模拟发邮件耗时操作
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info("给用户 [{}] 发送邮件...", username);
        });
        thread.start();
    }
}

@Slf4j
@Service
public class SimpleCouponService {

    ExecutorService executorService = Executors.newSingleThreadExecutor();
	// 线程池执行任务，减少资源消耗
    public void addCoupon(String username) {
        executorService.execute(() -> {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info("给用户 [{}] 发放优惠券", username);
        });
    }
}
```



这里用户注册事件对【发送短信和优惠券】其实是一对多的关系，可以使用观察者模式进行解耦：

```java
/**
 * 主题接口
 * @author Summerday
 */
public interface Subject {

    void registerObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers(String message);
}

/**
 * 观察者接口
 * @author Summerday
 */
public interface Observer {

    void update(String message);
}

@Component
@Slf4j
public class EmailObserver implements Observer {

    @Override
    public void update(String message) {
        log.info("向[{}]发送邮件", message);
    }
}
@Component
@Slf4j
public class CouponObserver implements Observer {

    @Override
    public void update(String message) {
        log.info("向[{}]发送优惠券",message);
    }
}

@Component
public class UserRegisterSubject implements Subject {

    @Autowired
    List<Observer> observers;

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(String username) {
        for (Observer observer : observers) {
            observer.update(username);
        }
    }
}


@RestController
@RequestMapping("/")
public class UserController {

    @Autowired
    UserRegisterSubject subject;

    @Autowired
    private SimpleUserService userService;


    @GetMapping("/reg")
    public String reg(String username) {
        userService.register(username);
        subject.notifyObservers(username);
        return "success";
    }
}
```

## 发布订阅模式是什么？

观察者模式和发布订阅模式是有一点点区别的，区别有以下几点：

- 前者：观察者订阅主题，主题也维护观察者的记录，而后者：**发布者和订阅者不需要彼此了解**，而是在消息队列或代理的帮助下通信，实现松耦合。
- 前者主要以同步方式实现，即某个事件发生时，由Subject调用所有Observers的对应方法，后者则主要使用消息队列异步实现。

> 图源：https://hackernoon.com/observer-vs-pub-sub-pattern-50d3b27f838c

![1_NcicKEqwUaI8VEc-Ejk6Dg](img/SpringBoot事件监听机制/1_NcicKEqwUaI8VEc-Ejk6Dg.jpeg)

尽管两者存在差异，但是他们其实在概念上相似，网上说法很多，不需要过于纠结，重点在于我们需要他们为什么出现，解决了什么问题。

## Spring事件监听机制概述

SpringBoot中事件监听机制则通过发布-订阅实现，主要包括以下三部分：

- 事件 ApplicationEvent，继承JDK的EventObject，可自定义事件。
- 事件发布者 ApplicationEventPublisher，负责事件发布。
- 事件监听者 ApplicationListener，继承JDK的EventListener，负责监听指定的事件。

我们通过SpringBoot的方式，能够很容易实现事件监听，接下来我们改造一下上面的案例：

## SpringBoot事件监听

### 定义注册事件

```java
public class UserRegisterEvent extends ApplicationEvent {
    
    private String username;

    public UserRegisterEvent(Object source) {
        super(source);
    }

    public UserRegisterEvent(Object source, String username) {
        super(source);
        this.username = username;
    }

    public String getUsername() {
        return username;
    }
}
```

### 注解方式 @EventListener定义监听器

```java
/**
 * 注解方式 @EventListener
 * @author Summerday
 */
@Service
@Slf4j
public class CouponService {
    /**
     * 监听用户注册事件,执行发放优惠券逻辑
     */
    @EventListener
    public void addCoupon(UserRegisterEvent event) {
        log.info("给用户[{}]发放优惠券", event.getUsername());
    }
}
```

### 实现ApplicationListener的方式定义监听器

```java
/**
 * 实现ApplicationListener<Event>的方式
 * @author Summerday
 */
@Service
@Slf4j
public class EmailService implements ApplicationListener<UserRegisterEvent> {
    /**
     * 监听用户注册事件, 异步发送执行发送邮件逻辑
     */
    @Override
    @Async
    public void onApplicationEvent(UserRegisterEvent event) {
        log.info("给用户[{}]发送邮件", event.getUsername());
    }
}
```

### 注册事件发布者

```java
@Service
@Slf4j
public class UserService implements ApplicationEventPublisherAware {

    // 注入事件发布者
    private ApplicationEventPublisher applicationEventPublisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

    /**
     * 发布事件
     */
    public void register(String username) {
        log.info("执行用户[{}]的注册逻辑", username);
        applicationEventPublisher.publishEvent(new UserRegisterEvent(this, username));
    }
}
```

### 定义接口

```java
@RestController
@RequestMapping("/event")
public class UserEventController {

    @Autowired
    private UserService userService;

    @GetMapping("/register")
    public String register(String username){
        userService.register(username);
        return "恭喜注册成功!";
    }
}                                                                                                                                                                                                                                                                                                                            
```

### 主程序类

```java
@EnableAsync // 开启异步
@SpringBootApplication
public class SpringBootEventListenerApplication {

    public static void main(String[] args) {

        SpringApplication.run(SpringBootEventListenerApplication.class, args);
    }

}
```

### 测试接口

启动程序，访问接口：`http://localhost:8081/event/register?username=天乔巴夏`，结果如下：

```java
2020-12-21 00:59:46.679  INFO 12800 --- [nio-8081-exec-1] com.hyh.service.UserService              : 执行用户[天乔巴夏]的注册逻辑
2020-12-21 00:59:46.681  INFO 12800 --- [nio-8081-exec-1] com.hyh.service.CouponService            : 给用户[天乔巴夏]发放优惠券
2020-12-21 00:59:46.689  INFO 12800 --- [         task-1] com.hyh.service.EmailService             : 给用户[天乔巴夏]发送邮件
```

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [芋道 Spring Boot 事件机制 Event 入门](http://www.iocoder.cn/Spring-Boot/Event/)
- [观察者模式和发布订阅模式有什么不同？](https://www.zhihu.com/question/23486749)
- [观察者模式](https://www.runoob.com/design-pattern/observer-pattern.html)
- [设计模式之美：观察者模式](https://time.geekbang.org/column/article/210170)
- [Observer vs Pub-Sub pattern](https://hackernoon.com/observer-vs-pub-sub-pattern-50d3b27f838c)

