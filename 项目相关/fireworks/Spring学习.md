Spring框架：轻量级开发框架，旨在提高开发人员的开发效率和系统的可维护性。

我们一般说的是SpringFrameWork ，是许多模块的集合，如核心core，数据访问，测试，web。

- **核心技术** ：依赖注入(DI)，AOP，事件(events)，资源，i18n，验证，数据绑定，类型转换，SpEL。
- **测试** ：模拟对象，TestContext框架，Spring MVC 测试，WebTestClient。
- **数据访问** ：事务，DAO支持，JDBC，ORM，编组XML。
- **Web支持** : Spring MVC和Spring WebFlux Web框架。
- **集成** ：远程处理，JMS，JCA，JMX，电子邮件，任务，调度，缓存。
- **语言** ：Kotlin，Groovy，动态语言。



# IoC

控制反转，将原本在程序中手动创建对象的控制权，交由Spring框架来管理。

Ioc容器就像是一个工厂，当我们需要创建一个对象的时候，只需要通过配置或注解注入即可，我们不需要再关心对象是如何被创建出来的，极大程度简化了应用中的复杂依赖。

SpringIoc过程：

xml->读取->Resource->解析->BeanDefinition->注册->BeanFactory

# Aop

Aspect-Oriented Programming:面向切面编程

将那些与业务无关，却为业务模块共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块之间的耦合度，利于扩展。

事务处理，日志管理，权限控制。

SpringAop基于动态代理，如果要代理的对象实现了某个接口，使用JDK 的Proxy去创建代理对象，如果没有实现接口，这时使用cglib，生成一个被代理对象的子类作为代理。

# SpringAop和AspectJAop

前者是运行时增强，后者是编译时增强。

前者基于代理，后者基于字节码操作。

SpringAop已经集成了AspectJ，如果切面过多，应当选择aspectj

# Bean的作用域

singleton：唯一bean实例，默认单例。

prototype：每次请求都会创建一个新的实例。

request：每一次HTTP请求都会产生一个新的实例，仅在当前request有效。

session：每一次HTTP请求都会产生一个新的实例，仅在当前session有效。

global-session：全局session作用域，仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话

# Spring中的单例bean线程安全问题

多个线程操作同一个对象的时候，对这个对象的非静态成员变量的写操作会存在线程安全问题。

- 在bean对象中尽量避免定义可变的成员变量。
- 在类中定义一个Threadlocal成员变量，将需要的可变变量保存在ThreadLocal中。

# Spring中的bean周期

- bean容器找到配置文件中的Spring bean的定义。
- bean容器利用反射创建一个Bean的实例。
- 如果涉及到一些属性值，利用set方法设置一些属性值。
- 如果 Bean 实现了 `BeanNameAware` 接口，调用 `setBeanName()`方法，传入Bean的名字。
- 如果 Bean 实现了 `BeanClassLoaderAware` 接口，调用 `setBeanClassLoader()`方法，传入 `ClassLoader`对象的实例。
- 如果Bean实现了 `BeanFactoryAware` 接口，调用 `setBeanClassLoader()`方法，传入 `ClassLoade` r对象的实例。
- 与上面的类似，如果实现了其他 `*.Aware`接口，就调用相应的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessBeforeInitialization()` 方法
- 如果Bean实现了`InitializingBean`接口，执行`afterPropertiesSet()`方法。
- 如果 Bean 在配置文件中的定义包含 init-method 属性，执行指定的方法。
- 如果有和加载这个 Bean的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessAfterInitialization()` 方法
- 当要销毁 Bean 的时候，如果 Bean 实现了 `DisposableBean` 接口，执行 `destroy()` 方法。
- 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 destroy-method 属性，执行指定的方法。

# SpringMVC的理解

MVC是一种设计模式，而SpringMVC是一款优秀的MVC框架，帮助我们进行更加简洁的web层开发，它天生与Spring框架集成。

controller层，service层，dao层，entity层。

![](https://mmbiz.qpic.cn/mmbiz_png/iaIdQfEric9TxiaKwgUUHQX0aVpNnuopm5wvNYJUuopN5R9BDcW2wxDdnEVyzke9a2paria5AcekHmVaFxicu6qV4kA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 客户端（浏览器）发送请求，直接请求到 `DispatcherServlet`。
2. `DispatcherServlet` 根据请求信息调用 `HandlerMapping`，解析请求对应的 `Handler`。
3. 解析到对应的 `Handler`（也就是我们平常说的 `Controller` 控制器）后，开始由 `HandlerAdapter` 适配器处理。
4. `HandlerAdapter` 会根据 `Handler`来调用真正的处理器开处理请求，并处理相应的业务逻辑。
5. 处理器处理完业务后，会返回一个 `ModelAndView` 对象，`Model` 是返回的数据对象，`View` 是个逻辑上的 `View`。
6. `ViewResolver` 会根据逻辑 `View` 查找实际的 `View`。
7. `DispaterServlet` 把返回的 `Model` 传给 `View`（视图渲染）。
8. 把 `View` 返回给请求者（浏览器）

# Spring框架中运用到的设计模式

- 工厂模式：Spring使用BeanFactory、ApplicaitonContext创建bean对象。
- 代理模式：spring aop功能的实现。
- 单例设计模式：spring中的bean默认都是单例的。
- 模板方法模式：jdbctemplate。
- 观察者模式：Spring事件驱动模型。
- 适配器模式：适配controller，aop的增强或通知
- 装饰器模式：根据需求动态切换不同的数据源。

# @Component和@Bean

1. 作用对象不同: `@Component` 注解作用于类，而`@Bean`注解作用于方法。
2. `@Component`通常是通过类路径扫描来自动侦测以及自动装配到Spring容器中（我们可以使用 `@ComponentScan` 注解定义要扫描的路径从中找出标识了需要装配的类自动装配到 Spring 的 bean 容器中）。`@Bean` 注解通常是我们在标有该注解的方法中定义产生这个 bean,`@Bean`告诉了Spring这是某个类的示例，当我需要用它的时候还给我。
3. `@Bean` 注解比 `Component` 注解的自定义性更强，而且很多地方我们只能通过 `@Bean` 注解来注册bean。比如当我们引用第三方库中的类需要装配到 `Spring`容器时，则只能通过 `@Bean`来实现。

`@Bean`注解使用示例：

```
@Configurationpublic class AppConfig {    @Bean    public TransferService transferService() {        return new TransferServiceImpl();    }}
```

上面的代码相当于下面的 xml 配置

```
<beans>    <bean id="transferService" class="com.acme.TransferServiceImpl"/></beans>
```

下面这个例子是通过 `@Component` 无法实现的。

```
@Beanpublic OneService getService(status) {    case (status)  {        when 1:                return new serviceImpl1();        when 2:                return new serviceImpl2();        when 3:                return new serviceImpl3();    }}
```

# Spring声明式事务

## Spring 事务中的隔离级别有哪几种?

**TransactionDefinition 接口中定义了五个表示隔离级别的常量：**

- **TransactionDefinition.ISOLATION_DEFAULT:** 使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **TransactionDefinition.ISOLATION_READ_COMMITTED:** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **TransactionDefinition.ISOLATION_REPEATABLE_READ:** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **TransactionDefinition.ISOLATION_SERIALIZABLE:** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

## Spring 事务中哪几种事务传播行为?

**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

## @Transactional(rollbackFor = Exception.class)注解了解吗？

我们知道：Exception分为运行时异常RuntimeException和非运行时异常。事务管理对于企业应用来说是至关重要的，即使出现异常情况，它也可以保证数据的一致性。

当`@Transactional`注解作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，同时，我们也可以在方法级别使用该标注来覆盖类级别的定义。如果类或者方法加了这个注解，那么这个类里面的方法抛出异常，就会回滚，数据库里面的数据也会回滚。

在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事物只会在遇到`RuntimeException`的时候才会回滚,加上`rollbackFor=Exception.class`,可以让事物在遇到非运行时异常时也回滚。



