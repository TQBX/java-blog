# JavaBlog

> Java 后端开发 技术知识总结

## Java基础

- [Java基础部分总结](Java基础/)

> 数据类型、面向对象、网络编程、反射、Java8、IO、JDBC、常用API、枚举

## MySQL专题

- [SQL必知必会](数据库部分/SQL必知必会/)

> DBMS、DDL、DML、DQL、sql函数、过滤条件、聚集函数、子查询、SQL92和SQL99、视图、存储过程、事务处理、事务隔离、游标

- [MySQL实战45讲](数据库部分/MySQL实战45讲/)

> SQL查询语句和更新语句执行原理、事务隔离、索引、锁机制

## 源码解析专题

### 集合源码系列

- [Java小白的源码学习系列：ArrayList](%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/md/ArrayList%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0.md)
- [Java小白的源码学习系列：LinkedList ](%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/md/LinkedList%20%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0.md)
- [Java小白的源码学习系列：Vector](%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/md/Vector%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0.md)
- [Java小白的源码学习系列：HashMap](%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/md/Hashmap%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0.md)
- [小白学Java：奇怪的RandomAccess](%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/md/%E5%B0%8F%E7%99%BD%E5%AD%A6Java%EF%BC%9A%E5%A5%87%E6%80%AA%E7%9A%84RandomAccess.md)
- [小白学Java：迭代器原来是这么回事](%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/md/%E5%B0%8F%E7%99%BD%E5%AD%A6Java%EF%BC%9A%E8%BF%AD%E4%BB%A3%E5%99%A8%E5%8E%9F%E6%9D%A5%E6%98%AF%E8%BF%99%E4%B9%88%E5%9B%9E%E4%BA%8B.md)

### Spring系列

- [源码：Spring容器的启动全流程+bean生命周期](面试相关/框架/Spring/Spring容器的启动全流程+bean生命周期.md)
- [源码：SpringAop源码解析+专业术语详解](面试相关/框架/Spring/Aop源码解析+专业术语详解.md)
- [源码：Spring的三级缓存解决循环依赖](面试相关/框架/Spring/Spring的三级缓存解决循环依赖.md)
- [源码：过滤器和拦截器](面试相关/框架/Spring/过滤器和拦截器.md)
- [源码：SpringMVC运行流程](框架系列/springmvc/md/springMVC运行流程)

### Mybatis系列

- [MyBatis源码之根据配置文件创建sqlsessionfactory](框架系列/mybatis/mybatis源码之根据配置文件创建sqlsessionfactory.md)
- [MyBatis源码之创建sqlSession实现类](框架系列/mybatis/mybatis源码之创建sqlSession实现类.md)
- [MyBatis源码之getMapper返回代理对象](框架系列/mybatis/mybatis源码之getMapper返回代理对象.md)
- [MyBatis源码之基于动态代理查询](框架系列/mybatis/mybatis源码之基于动态代理查询.md)
- [MyBatis源码之查询过程](框架系列/mybatis/mybatis源码之查询过程.md)
- [MyBatis源码之一级缓存与二级缓存](框架系列/mybatis/mybatis源码之缓存设置.md)
- [MyBatis源码之自定义插件开发](框架系列/mybatis/mybatis源码之自定义插件开发.md)

### 并发系列

- [Java并发包源码学习系列：ThreadLocal类源码解析](面试相关/并发/ThreadLocal.md)
- [Java并发包源码学习系列：AbstractQueuedSynchronizer](多线程并发/md/AQS.md)
- [Java并发包源码学习系列：CLH同步队列及同步资源获取与释放](多线程并发/md/waitQueue.md)
- [Java并发包源码学习系列：AQS共享式与独占式获取与释放资源的区别](多线程并发/md/exlusive-and-share.md)
- [Java并发包源码学习系列：ReentrantLock可重入独占锁详解](多线程并发/md/reentrantLock.md)
- [Java并发包源码学习系列：ReentrantReadWriteLock读写锁解析](多线程并发/md/read_write_lock.md)
- [Java并发包源码学习系列：详解Condition条件队列、signal和await](多线程并发/md/condition.md)
- [Java并发包源码学习系列：挂起与唤醒线程LockSupport工具类](多线程并发/md/LockSupport.md)
- [Java并发包源码学习系列：JDK1.8的ConcurrentHashMap源码解析](多线程并发/md/concurrent_hashmap.md)
- [Java并发包源码学习系列：阻塞队列BlockingQueue及实现原理分析](多线程并发/md/blockingQueue.md)
- [Java并发包源码学习系列：阻塞队列实现之ArrayBlockingQueue源码解析](多线程并发/md/array-blocking-queue.md)
- [Java并发包源码学习系列：阻塞队列实现之LinkedBlockingQueue源码解析](多线程并发/md/blocking-queue-linked.md)
- [Java并发包源码学习系列：阻塞队列实现之PriorityBlockingQueue源码解析](多线程并发/md/blocking-queue-priority.md)
- [Java并发包源码学习系列：阻塞队列实现之DelayQueue源码解析](多线程并发/md/blocking-queue-delay.md)
- [Java并发包源码学习系列：阻塞队列实现之SynchronousQueue源码解析](多线程并发/md/blocking-queue-sync.md)
- [Java并发包源码学习系列：阻塞队列实现之LinkedTransferQueue源码解析](多线程并发/md/blocking-queue-linked-transfer.md )
- [Java并发包源码学习系列：阻塞队列实现之LinkedBlockingDeque源码解析](多线程并发/md/blocking-queue-deque.md)
- [Java并发包源码学习系列：基于CAS非阻塞并发队列ConcurrentLinkedQueue源码解析](多线程并发/md/concurentLinkedQueue.md)
- [Java并发包源码学习系列：线程池ThreadPoolExecutor源码解析](多线程并发/md/ThreadPoolExecutor.md)
- [Java并发包源码学习系列：线程池ScheduledThreadPool源码解析](多线程并发/md/ScheduledThreadPool.md)
- [Java并发包源码学习系列：同步组件CountDownLatch源码解析](多线程并发/md/count_down_latch.md)
- [Java并发包源码学习系列：同步组件CyclicBarrier源码解析](多线程并发/md/cyclic_barrier.md)
- [Java并发包源码学习系列：同步组件Semaphore源码解析](多线程并发/md/semaphore.md)

## Java并发编程

- [【Java并发编程】常见基础问题整理](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/%E5%B9%B6%E5%8F%91%E5%9F%BA%E7%A1%80%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98.md)
- [【Java并发编程】线程的六种状态及转化](%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91/md/Java%EF%BC%9A%E7%BA%BF%E7%A8%8B%E7%9A%84%E5%85%AD%E7%A7%8D%E7%8A%B6%E6%80%81%E5%8F%8A%E8%BD%AC%E6%8D%A2.md)
- [【Java并发编程】谈谈控制线程的几种办法](%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91/md/Java%EF%BC%9A%E8%B0%88%E8%B0%88%E6%8E%A7%E5%88%B6%E7%BA%BF%E7%A8%8B%E7%9A%84%E5%87%A0%E7%A7%8D%E5%8A%9E%E6%B3%95.md)
- [【Java并发编程】谈谈如何实现线程间正确通信](%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91/md/Java%EF%BC%9A%E5%AE%9E%E7%8E%B0%E7%BA%BF%E7%A8%8B%E9%97%B4%E6%AD%A3%E7%A1%AE%E9%80%9A%E4%BF%A1.md)
- [【Java并发编程】谈谈线程的互斥与同步](%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91/md/Java%EF%BC%9A%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E4%B8%8E%E4%BA%92%E6%96%A5%E5%90%8C%E6%AD%A5.md)
- [【Java并发编程】Lock与ReetrantLock重入锁 的特点](%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%B9%B6%E5%8F%91/md/Java%EF%BC%9ALock%E4%B8%8EReentrantLock.md)
- [【Java并发编程】线程池相关知识点整理](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9.md)
- [【Java并发编程】synchronized相关面试题总结](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/synchronized.md)
- [【Java并发编程】从CPU缓存模型到JMM来理解volatile关键字](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/volatile%E5%85%B3%E9%94%AE%E5%AD%97.md)
- [【Java并发编程】并发操作原子类Atomic以及CAS的ABA问题](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/Atomic%E5%8E%9F%E5%AD%90%E7%B1%BB.md)
- [【Java并发编程】CountDownLatch，CyclicBarrier，Semphore，Exchanger](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/Java%E4%B8%AD%E7%9A%84%E5%B9%B6%E5%8F%91%E5%B7%A5%E5%85%B7%E7%B1%BB.md)
- [【Java并发编程】面试常考的ThreadLocal，超详细源码学习](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/%E5%B9%B6%E5%8F%91/ThreadLocal.md)



## 计算机网络+操作系统

- [【考研】计算机网络复习重点！](计算机网络/复习重点.md)
- [计算机网络知识点总结](面试相关/计算机网络/计算机网络知识点总结.md)

- [【考研】操作系统复习重点！](操作系统/知识点总结.md)

## JVM系列

- [JVM内存区域](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/JVM/jvm%E5%86%85%E5%AD%98%E5%8C%BA%E5%9F%9F.md)
- [四种引用类型的特点](%E9%9D%A2%E8%AF%95%E7%9B%B8%E5%85%B3/JVM/%E5%9B%9B%E7%A7%8D%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%E7%9A%84%E7%89%B9%E7%82%B9.md)
- [垃圾收集器与内存分配策略](JVM%E7%B3%BB%E5%88%97/%E5%9E%83%E5%9C%BE%E6%94%B6%E9%9B%86%E5%99%A8%E4%B8%8E%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5.md)
- 类文件结构
- [虚拟机类加载机制](JVM%E7%B3%BB%E5%88%97/%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6.md)
- [JDK监控和故障处理工具](面试相关/JVM/JDK监控和故障处理工具.md)

## Spring与SpringMVC

- [Spring入门介绍](框架系列/Spring/md/Spring入门)
- [SpringMVC如何介绍Ajax的json参数](框架系列/springmvc/md/接收ajaxjson参数.md)
- [SpringMVC如何实现文件上传](框架系列/springmvc/md/文件上传.md)
- [SpringMVC如何支持Restful风格](框架系列/springmvc/md/支持restful风格.md)
- [SpringMVC如何自定义视图和视图解析](框架系列/springmvc/md/自定义视图和视图解析.md)
- [SpringMVC如何介绍Ajax的json参数](框架系列/springmvc/md/SpringMVC国际化操作.md)
- [SpringMVC的国际化操作](框架系列/springmvc/md/接收ajaxjson参数.md)
- [SpringMVC异常处理器入门](框架系列/springmvc/md/异常处理器入门.md)
- [SpringMVC拦截器入门](框架系列/springmvc/md/拦截器入门.md)
- [SpringMVC入门踩坑集合](框架系列/springmvc/md/入门踩坑集合.md)

## SpringBoot专题

- [SpringBoot](框架系列/springboot/SpringBoot/SpringBoot.md)
- [SpringBoot快速入门](框架系列/springboot/md/Springboot基础知识学习.md)
- [SpringBoot与FreeMarker整合](框架系列/springboot/md/FreeMarker使用及SpringBoot整合.md)
- [SpringBoot中Logback日志配置解析](框架系列/springboot/md/SpringBoot中Logback日志配置解析.md)
- [SpringBoot中整合权限管理框架shiro](框架系列/springboot/md/SpringBoot中整合权限管理框架shiro.md)
- [SpringBoot中的异步任务](框架系列/springboot/md/SpringBoot中的异步任务.md)
- [SpringBoot中的配置解析【Externalized Configuration】](框架系列/springboot/md/SpringBoot中配置解析.md)
- [SpringBoot利用Aop巧妙打印日志信息](框架系列/springboot/md/SpringBoot利用Aop巧妙打印日志信息.md)
- [SpringBoot在IDEA中实现热部署](框架系列/springboot/md/SpringBoot在IDEA中实现热部署.md)
- [SpringBoot整合Docker快速部署](框架系列/springboot/md/SpringBoot整合Docker快速部署.md)
- [SpringBoot整合H2数据库](框架系列/springboot/md/SpringBoot整合H2数据库.md)
- [SpringBoot整合MybatisPlus](框架系列/springboot/md/SpringBoot整合MybatisPlus.md)
- [SpringBoot整合Thymeleaf](框架系列/springboot/md/SpringBoot整合Thymeleaf.md)
- [SpringBoot整合swagger-ui快速生成在线接口文档](框架系列/springboot/md/SpringBoot整合swagger-ui快速生成在线接口文档.md)
- [SpringBoot文件上传](框架系列/springboot/md/SpringBoot文件上传.md)
- [SpringBoot瘦身部署](框架系列/springboot/md/SpringBoot瘦身部署.md)
- [SpringBoot优雅的参数校验BeanValidation](框架系列/springboot/md/SpringBoot的参数校验.md)
- [SpringBoot统一异常处理](框架系列/springboot/md/SpringBoot统一异常处理.md)
- [SpringBoot项目部署到Linux服务器全流程](框架系列/springboot/md/SpringBoot项目部署到Linux服务器全流程.md)
- [SpringBoot整合Spring Data JPA](框架系列/springboot/md/SpringBoot中使用JPA.md)
- [SpringBoot中JPA的过滤查询Specifications](框架系列/springboot/md/SpringBoot中JPA的过滤查询Specifications.md)
- [SpringBoot与定时任务](框架系列/springboot/md/SpringBoot定时任务.md)
- [SpringBoot使用SpringDataREST快速构建restful应用](框架系列/springboot/md/SpringBoot使用SpringDataREST快速构建restful应用.md)
- [SpringBoot整合任务调度框架Quartz](框架系列/springboot/md/SpringBoot整合任务调度框架Quartz.md)
- [SpringBoot事件监听机制](框架系列/springboot/md/SpringBoot事件监听机制.md)
- [SpringBoot使用Actuator服务监控](框架系列/springboot/md/SpringBoot使用Actuator服务监控.md)


## SpringCloud专题

- [SpringCloud学习笔记（零）简介：官方文档翻译](微服务学习/SpringCloud学习笔记（零）简介：官方文档翻译.md)
- [SpringCloud学习笔记（一）基础环境搭建](微服务学习/SpringCloud学习笔记（一）基础环境搭建.md)
- [SpringCloud学习笔记（二）Eureka服务注册与发现](微服务学习/SpringCloud学习笔记（二）Eureka服务注册与发现.md)
- [SpringCloud学习笔记（三）集群信息获取+服务列表信息获取](微服务学习/SpringCloud学习笔记（三）集群信息获取+服务列表信息获取.md)
- [SpringCloud学习笔记（四）Eureka的自我保护机制](微服务学习/SpringCloud学习笔记（四）Eureka的自我保护机制.md)
- [SpringCloud学习笔记（五）Zookeeper服务注册与发现](微服务学习/SpringCloud学习笔记（五）Zookeeper服务注册与发现.md)
- [SpringCloud学习笔记（六）Consul服务注册与发现](微服务学习/SpringCloud学习笔记（六）Consul服务注册与发现.md)
- [SpringCloud学习笔记（七）Eureka，Consul，Zookeeper三个注册中心异同点](微服务学习/SpringCloud学习笔记（七）三个注册中心异同点.md)
- [SpringCloud学习笔记（八）Ribbon负载均衡服务调用](微服务学习/SpringCloud学习笔记（八）Ribbon负载均衡.md)
- [SpringCloud学习笔记（九）OpenFeign服务调用](微服务学习/SpringCloud学习笔记（九）OpenFeign服务调用.md)
- [SpringCloud学习笔记（十）Hystrix降级、熔断和图形化监控](微服务学习/SpringCloud学习笔记（十）Hystrix断路器.md)
- [SpringCloud学习笔记（十一）Gateway网关](微服务学习/SpringCloud学习笔记（十一）服务网关Gateway.md)
- [SpringCloud学习笔记（十二）Spring Cloud Config服务配置](微服务学习/SpringCloud学习笔记（十二）SpringCloudConfig服务配置.md)
- [SpringCloud学习笔记（十三）Spring Cloud Bus消息总线](微服务学习/SpringCloud学习笔记（十三）SpringCloudBus消息总线.md)
- [SpringCloud学习笔记（十四）Spring Cloud Stream消息驱动](微服务学习/SpringCloud学习笔记（十四）SpringCloudStream消息驱动.md)
- [SpringCloud学习笔记（十五）Spring Cloud Sleuth链路追踪](微服务学习/SpringCloud学习笔记（十五）SpringCloudSleuth链路追踪.md)
- [SpringCloud Alibaba学习笔记：入门简介](微服务学习/SpringCloudAlibaba学习笔记：入门简介.md)
- [SpringCloud Alibaba学习笔记：Nacos服务注册发现](微服务学习/SpringCloudAlibaba学习笔记：Nacos服务注册发现.md)
- [SpringCloud Alibaba学习笔记：Nacos服务配置中心](微服务学习/SpringCloudAlibaba学习笔记：Nacos服务配置中心.md)
- [SpringCloud Alibaba学习笔记：Nacos集群和持久化配置](微服务学习/SpringCloudAlibaba学习笔记：Nacos集群和持久化配置.md)

## MyBatis专题


- [MyBatis入门](框架系列/mybatis/mybatis入门.md)
- [MyBatis配置](框架系列/mybatis/mybatis配置.md)
- [MyBatis动态SQL语句](框架系列/mybatis/动态SQL语句.md)
- [MyBatis官网学习settings](框架系列/mybatis/官网学习settings.md)
- [MyBatis部分用法标签总结](框架系列/mybatis/mybatis部分用法标签总结.md)
- [MyBatis多表查询](框架系列/mybatis/mybatis多表查询.md)
- [MyBatis延迟加载](框架系列/mybatis/mybatis延迟加载.md)
- [MyBatis整合第三方缓存](框架系列/mybatis/mybatis整合第三方缓存.md)
- [MyBatis逆向工程配置](框架系列/mybatis/mabatis逆向工程配置.md)
- [MyBatis查漏补缺](框架系列/mybatis/mybatis查漏补缺.md)

## Linux专题

- [目录结构及常用命令](linux学习系列/常用命令及目录结构.md)
- [常用命令查漏补缺](linux学习系列/命令查漏补缺.md)
- [用户及用户组的管理](linux学习系列/用户及用户组的管理.md)
- [Linux文件的基本属性](linux学习系列/Linux文件的基本属性.md)
- [修改运行级别.md](linux学习系列/运行级别.md)
- [设置CentOS网卡开机自启](linux学习系列/设置CentOS网卡开机自启.md)
- [搭建JavaEE环境](linux学习系列/搭建JavaEE环境.md)
- [Linux系统部署nginx](linux学习系列/Linux系统部署nginx.md)
- [vi和vim](linux学习系列/vi和vim.md)

## Docker专题

- [docker踩坑记录](linux学习系列/docker踩坑记录.md)
- [docker踩坑第二弹](linux学习系列/docker踩坑第二弹.md)
- [docker安装kibana](linux学习系列/docker安装kibana.md)
- [docker安装Zookeeper](linux学习系列/docker安装zookeeper)
- [docker安装Consul](linux学习系列/docker安装consul)

## 力扣刷题系列

- [1 - 10题题解](力扣刷题系列/1-100/1-10.md)
- [11 - 20题题解](力扣刷题系列/1-100/11-20.md)
- [21 - 30题题解](力扣刷题系列/1-100/21-30.md)
- [31 - 40题题解](力扣刷题系列/1-100/31-40.md)
- [41 - 50题题解](力扣刷题系列/1-100/41-50.md)
- [51 - 60题题解](力扣刷题系列/1-100/51-60.md)
- [61 - 70题题解](力扣刷题系列/1-100/61-70.md)
- [71 - 80题题解](力扣刷题系列/1-100/71-80.md)

## 工具使用

- [教你如何使用docsify快速部署优美的在线文档](others/docsify.md)
- [版本管理工具git的使用](工具集合/git/git使用.md)

## 中间件

- [分布式消息处理kafka](中间件/kafka.md)
- [任务调度框架Quartz](中间件/Quartz.md)
- [Zookeeper](中间件/zookeeper)
- nginx

## 【就是学！】

[面试相关](面试相关/)

# 杂

- [JWT](others/jwt.md)

- [考研备考经验分享](others/learn.md)
- [密码学期末复习笔记](others/passwd.md)

