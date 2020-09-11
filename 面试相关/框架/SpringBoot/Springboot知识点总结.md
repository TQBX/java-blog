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
  - 由父ApplicationContext加载，优先于application.yml加载。
  - boostrap中的属性无法被覆盖。
- application.yml或application.propertiees
  - 常用，用于SpringBoot的自动化配置。

