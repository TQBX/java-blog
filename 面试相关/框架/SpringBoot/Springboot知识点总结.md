# SpringBoot的优点

SpringBoot官网：[https://spring.io/projects/spring-boot](https://spring.io/projects/spring-boot)

> - Create **stand-alone** Spring applications
> - **Embed** Tomcat, Jetty or Undertow directly (no need to deploy WAR files)
> - **Provide opinionated 'starter' dependencies** to simplify your build configuration
> - **Automatically configure** Spring and 3rd party libraries whenever possible
> - **Provide production-ready features** such as metrics, health checks, and externalized configuration
> - Absolutely **no code generation and no requirement for XML configuration**

- 可以创建独立运行的Spring应用。
- 内嵌了各种Servlet容器，如Tomcat，Jetty等，不需要打成war包部署到容器中，只需要打成可执行的jar就能独立运行，所有依赖包都在一个jar内。
- 提供了许多特定场景下的starter依赖，以简化配置。
- 自动配置Spring，根据类路径下的类，jar自动配置bean。
- 提供了应用监控的功能。
- 无代码生成和XML配置。

# SpringBoot的两种配置文件

- bootstrap.yml或bootstrap.properties
  - 由父ApplicationContext加载，**优先于application.yml加载**。
  - boostrap中的属性无法被覆盖。
- application.yml或application.propertiees
  - 常用，用于SpringBoot的自动化配置。
  - 一般用于定义单个应用级别。

# SpringBoot常用的starter

- spring-boot-starter-web - Web 和 RESTful 应用程序；
- spring-boot-starter-test - 单元测试和集成测试；
- spring-boot-starter-jdbc - 传统的 JDBC；
- spring-boot-starter-security - 使用 SpringSecurity 进行身份验证和授权；
- spring-boot-starter-data-jpa - 带有 Hibernate 的 Spring Data JPA；
- spring-boot-starter-data-rest - 使用 Spring Data REST 公布简单的 REST 服务

# SpringBoot核心注解

核心注解是@SpringBootApplication 由以下三种组成

- @SpringBootConfiguration：组合了 @Configuration 注解，实现配置文件的功能。
- @EnableAutoConfiguration：打开自动配置的功能。
- @ComponentScan：Spring组件扫描。

# SpringBoot启动流程

- new springApplication对象，利用spi机制加载applicationContextInitializer， applicationLister接口实例（META-INF/spring.factories）；

- 调run方法准备Environment，加载应用上下文（applicationContext），发布事件 很多通过listener实现

- 创建spring容器， refreshContext（） ，实现starter自动化配置，spring.factories文件加载， bean实例化

  #### SpringBoot自动配置的原理

  - @EnableAutoConfiguration找到META-INF/spring.factories（需要创建的bean在里面）配置文件
  - 读取每个starter中的spring.factories文件

