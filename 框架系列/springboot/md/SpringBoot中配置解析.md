[toc]

# SpringBoot中的配置解析【Externalized Configuration】

## 本篇要点

- 介绍各种配置方式的优先级。
- 介绍各种外部化配置方式。
- 介绍yaml的格式及原理。
- 介绍如何绑定并测试类型安全的属性配置。
- 介绍@ConfigurationProperties与@Value的区别。

## 一、SpringBoot官方文档对于外部化配置的介绍及作用顺序

SpringBoot支持多种外部化配置，以便于开发者能够在不同的环境下，使用同一套应用程序代码。外部化配置的方式有多种：properties文件，yaml文件，Environment变量已经命令行参数等等。

外部化配置的属性值可以通过@Value注解自动注入，亦可以通过Spring的Environment抽象访问，也可以通过@ConfigurationProperties注解绑定到结构化对象上。

SpringBoot支持很多种的外部化配置，待会我们会介绍到。在这之前，我们必须要知道如果多种配置同时出现，一定是按照特定的顺序生效的。规则如下：

1. devtool处于active状态时，`$HOME/.config/spring-boot`目录中的Devtool全局配置。
2. 测试中的@TestPropertySource注解。
3. 测试中的@SpringBootTest#properties注解特性。
4. 命令行参数。
5. `SPRING_APPLICATION_JSON`中的属性(环境变量或系统属性中的内联JSON嵌入)。
6. `ServletConfig`初始化参数。
7. `ServletContext`初始化参数。
8. `java:comp/env`里的JNDI属性
9. JVM系统属性`System.getProperties()`。
10. 操作系统环境变量
11. 仅具有`random.*`属性的`RandomValuePropertySource `。
12. 应用程序以外的application-{profile}.properties或者application-{profile}.yml文件
13. 打包在应用程序内的application-{profile}.properties或者application-{profile}.yml文件
14. 应用程序以外的application.properties或者appliaction.yml文件
15. 打包在应用程序内的application.properties或者appliaction.yml文件
16. @Configuration类上的@PropertySource注解，需要注意，在ApplicationContext刷新之前，是不会将这个类中的属性加到环境中的，像`logging.*,spring.main.*`之类的属性，在这里配置为时已晚。
17. 默认属性(通过`SpringApplication.setDefaultProperties`指定).

这里列表按组优先级排序，也就是说，**任何在高优先级属性源里设置的属性都会覆盖低优先级的相同属性**，列如我们上面提到的命令行属性就覆盖了application.properties的属性。

举个例子吧：

如果在application.properties中设置`name=天乔巴夏`，此时我用命令行设置`java -jar hyh.jar --author.name=summerday`，最终的name值将会是summerday，因为命令行属性优先级更高。

## 二、各种外部化配置举例

### 1、随机值配置

配置文件中`${random}` 可以用来生成各种不同类型的随机值，从而简化了代码生成的麻烦，例如 生成 int 值、long 值或者 string 字符串。原理在于，`RandomValuePropertySource`类重写了`getProperty`方法，判断以`random.`为前缀之后，进行了适当的处理。

```properties
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.lessThanTen=${random.int(10)}
my.inRange=${random.int[1024,65536]}
```

### 2、命令行参数配置

默认情况下，SpringApplication将所有的命令行选项参数【以`--`开头的参数，如`--server.port=9000`】转换为属性，并将它们加入SpringEnvironment中，命令行属性的配置始终优先于其他的属性配置。

如果你不希望将命令行属性添加到Environment中，可以使用`SpringApplication.setAddCommandLineProperties(false)`禁用它。

```shell
$ java -jar app.jar --debug=true #开启debug模式，这个在application.properties文件中定义debug=true是一样的
```

### 3、属性文件配置

属性文件配置这一部分是我们比较熟悉的了，我们在快速创建SpringBoot项目的时候，默认会在resources目录下生成一个application.properties文件。SpringApplication都会从配置文件加载配置的属性，并最终加入到Spring的Environment中。除了resources目录下，还有其他路径，SpringBoot默认是支持存放配置文件的。

1. 当前项目根目录下的 `/config` 目录下
2. 当前项目的根目录下
3. resources 目录下的 `/config` 目录下
4. resources 目录下

以上四个，优先级从上往下依次降低，也就是说，如果同时出现，上面配置的属性将会覆盖下面的。

> 关于配置文件，properties和yaml文件都能够满足配置的需求。

当然，这些配置都是灵活的，如果你不喜欢默认的配置文件命名或者默认的路径，你都可以进行配置：

```shell
$ java -jar myproject.jar --spring.config.name=myproject
$ java -jar myproject.jar --spring.config.location=classpath:/default.properties,classpath:/override.properties
```

### 4、指定profile属性

通常情况下，我们开发的应用程序需要部署到不同的环境下，属性的配置自然也需要不同。如果每次在发布的时候替换配置文件，过于麻烦。SpringBoot的多环境配置为此提供了便利。具体做法如下：

我们之前在介绍各种配置的优先级的时候说过，`application-{profile}.properties或者application-{profile}.yml文件`的优先级高于`application.properties或application.yml`配置，这里的profile就是我们定义的环境标识：

我们在resource目录下创建三个文件：

- application.properties：默认的配置，default。
- application-dev.properties：开发环境，dev。
- application-prod.properties：生产环境，prod。

我们可以通过指定`spring.profiles.active`属性来激活对应的配置环境：

```java
spring.profiles.active=dev
```

或使用命令行参数的配置形式：

```shell
$ java -jar hyh.jar --spring.profiles.active=dev
```

如果没有profile指定的文件于profile指定的文件的配置属性同时定义，那么指定profile的配置优先。

### 5、使用占位符

在使用application.properties中的值的时候，他们会从Environment中获取值，那就意味着，可以引用之前定义过的值，比如引用系统属性。具体做法如下：

```properties
name=天乔巴夏
description=${name} is my name
```

### 6、加密属性

Spring Boot不提供对加密属性值的任何内置支持，但是，它提供了**修改Spring环境中的值**所必需的挂钩点。我们可以通过实现EnvironmentPostProcessor接口在应用程序启动之前操纵Environment。

可以参考[howto.html](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-customize-the-environment-or-application-context)，查看具体使用方法。

### 7、使用YAML代替properties

YAML是JSON的超集，是一种**指定层次结构配置数据的便捷格式**，我们以properties文件对比一下就知道了：

```properties
#properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/test?serverTimezone=GMT%2B8
spring.datasource.username=root
spring.datasource.password=123456

my.servers[0]=www.hyh.com
my.servers[1]=www.yhy.com
```

```yml
# yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/test?serverTimezone=GMT%2B8
    username: root
    password: 123456
my:
  server:
    - www.hyh.com
    - www.yhy.com
```

只要在类路径上具有SnakeYAML库，SpringApplication类就会自动支持YAML作为属性配置的方式。SpringBoot项目中的`spring-boot-starter`已经提供了相关类库：`org.yaml.snakeyaml`，因此SpringBoot天然支持这种方式配置。

关于yaml文件的格式，可以参考官方文档：[Using YAML Instead of Properties](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-yaml)

### 8、类型安全的属性配置

上面说到通过`@Value("${property}") `注解来注入配置有时会比较麻烦，特别是当多个属性本质上具有层次结构的时候。SpringBoot提供了一种解决方案：**让强类型的bean管理和验证你的配置**。

直接来看具体的使用叭：

#### @ConfigurationPropertie定义一个绑定配置的JavaBean

1. 使用默认构造器+getter和setter注入

```java
@ConfigurationProperties("acme")
public class AcmeProperties {
    private boolean enabled; //acme.enabled  默认为false
    private InetAddress remoteAddress;// acme.remote-address  可以从String转换而来的类型
    private final Security security = new Security();
	//.. 省略getter和setter方法

    public static class Security {
        private String username; // acme.security.username
        private String password; // acme.security.password
        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));// acme.security.roles
		//.. 省略getter setter方法
    }
}
```

> 这种方式依赖于默认的空构造函数，通过getter和setter方法赋值，因此getter和setter方法是必要的，且不支持静态属性的绑定。
>
> 如果嵌套pojo属性已经被初始化值： `private final Security security = new Security();`可以不需要setter方法。如果希望绑定器使用其默认构造函数动态创建实例，则需要setter。

2. 通过@ContructorBinding注解使用构造器绑定的方式：

```java
@ConstructorBinding //标注使用构造器绑定
@ConfigurationProperties("acme")
public class AcmeProperties {
    private final Security security;
    private final boolean enabled;
    private final InetAddress remoteAddress;
    public AcmeProperties(boolean enabled, InetAddress remoteAddress, Security security) {
        this.enabled = enabled;
        this.remoteAddress = remoteAddress;
        this.security = security;
    }
    //..省略getter方法
    @ToString
    public static class Security {
        private final String username;
        private final String password;
        private final List<String> roles;
        public Security(String username, String password,
                        @DefaultValue("USER") List<String> roles) {
            this.username = username;
            this.password = password;
            this.roles = roles;
        }
    }
    //..省略getter方法
}
```

> 如果没有配置Security实例属性，那么最后结果：Security=null。如果我们想让Security={username=null,password=null,roles=[USER]}，可以在Security上加上@DefaultValue。`public AcmeProperties(boolean enabled, InetAddress remoteAddress, @DefaultValue Security security)`

#### 通过@EnableConfigurationProperties注册

已经定义好了JavaBean，并与配置属性绑定完成，接着需要注册这些bean。我们通常用的@Component或@Bean，@Import加载bean的方式在这里是不可取的，SpringBoot提供了解决方案：**使用@EnableConfigurationProperties**，我们既可以一一指定配置的类，也可以按照组件扫描的方式进行配置。

```java
@SpringBootApplication
@EnableConfigurationProperties({HyhConfigurationProperties.class, MyProperties.class,AcmeProperties.class})
public class SpringBootProfileApplication {
}
```

```java
@SpringBootApplication
@ConfigurationPropertiesScan({"com.hyh.config"})
public class SpringBootProfileApplication {
}
```

#### 配置yaml文件

```yml
acme:
  remote-address: 192.168.1.1
  security:
    username: admin
    roles:
      - USER
      - ADMIN
```

#### 注入properties，测试

```java
@Configuration
public class Application implements CommandLineRunner {
    @Autowired
    private AcmeProperties acmeProperties;

    @Override
    public void run(String... args) throws Exception {
        System.out.println(acmeProperties);
    }
}
```

```java
//输出： 
AcmeProperties(security=AcmeProperties.Security(username=admin, password=null, roles=[USER, ADMIN]), enabled=false, remoteAddress=/192.168.1.1)
```

#### 宽松绑定

SpringBoot采用宽松的规则进行Environment和@ConfigurationProperties标注bean的匹配。如：

```java
@ConfigurationProperties(prefix="acme.my-project.person")
public class OwnerProperties {

    private String firstName;

    public String getFirstName() {
        return this.firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

}
```

下面表格中的属性名都可以匹配：

| Property                            | Note                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| `acme.my-project.person.first-name` | **Kebab case**, which is recommended for use in `.properties` and `.yml` files. |
| `acme.myProject.person.firstName`   | **Standard camel case syntax**.                              |
| `acme.my_project.person.first_name` | **Underscore notation**, which is an alternative format for use in `.properties` and `.yml` files. |
| `ACME_MYPROJECT_PERSON_FIRSTNAME`   | **Upper case format**, which is recommended when using system environment variables. |

@ConfigurationProperties注解中的prefix值必须是kebab case形式的，以`-`为分割符。

> Spring官方建议，属性尽可能以lower-case kebab的形式：my.property-name=acme

**Map如何绑定**

绑定到Map属性时，如果key包含**小写字母数字字符或-以外的任何其他字符**，则需要使用方括号包围key，以便保留原始值。 如果键没有被`[]`包围，则所有非字母数字或-的字符都将被删除。如下：

```yml
hyh:
  username: 天乔巴夏
  password: 123456
  map:
    "[/key1]": value1 #用引号包围[]，用[]包围key
    /key3: value3
    key-4: value4
    key/5: value5
# 结果："map":{/key1=value1,key5=value5, key-4=value4, key3=value3}
```

**环境变量如何绑定**

遵循三条原则：

1. 把`.`换成下划线`_`。
2. 移除`-`。
3. 小写转大写。

如：`spring.main.log-startup-info`转为：`SPRING_MAIN_LOGSTARTUPINFO`，`my.acme[0].other`转为`MY_ACME_0_OTHER`。

### 9、复杂类型

之前介绍yml文件，介绍了单纯的数组形式或值的绑定，SpringBoot还支持复杂类型的绑定。

```yml
merge:
  list:
    - name: 天乔巴夏
      desc: 帅啊
    - name: tqbx
      desc: 很帅啊
  map:
    key1:
      name: summerday
      desc: handsome!
    key2:
      name: summer
```

```java
@ToString
@ConfigurationProperties(prefix = "merge")
public class MergeProperties {
    private final List<User> list = new ArrayList<>();
    private final Map<String,User> map = new HashMap<>();
    public List<User> getList() {
        return list;
    }
    public Map<String, User> getMap() {
        return map;
    }
}
```

最后输出：

```java
MergeProperties(
    list=[User(name=天乔巴夏, desc=帅啊), 
          User(name=tqbx, desc=很帅啊)], 
    map={key1=User(name=summerday, desc=handsome!), 
         key2=User(name=summer, desc=null)}a
)
```

### 10、参数校验

对@ConfigurationProperties类使用Spring的`@Valid`注解时，Spring Boot就会尝试对其进行验证。

你可以直接在配置类上使用JSR-303 `javax.validation`约束注解。这个做法的前提是，你的类路径上有兼容的JSR-303实现：

```xml
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-validator</artifactId>
            <version>6.0.18.Final</version>
        </dependency>
```

然后将约束注解加到字段上，如下：

```java
@Data
@Validated
@ConfigurationProperties(prefix = "validate")
public class ValidateProperties {
    @NotNull
    private String name;
    @Valid
    private final SubProperties subProperties = new SubProperties();
    @Data
    public static class SubProperties {
        @Min(value = 10,message = "年龄最小为10")
        public Integer age;
    }
}
```

配置如下：

```yml
validate:
  name: hyh
  sub-properties:
    age: 5
```

结果如下：

```java
Description:

Binding to target org.springframework.boot.context.properties.bind.BindException: 
Failed to bind properties under 'validate' to com.hyh.config.ValidateProperties failed:
    Property: validate.sub-properties.age
    Value: 5
    Origin: class path resource [application.yml]:47:10
    Reason: 年龄最小为10
        
Action:
Update your application's configuration
```

## 三、@ConfigurationProperties与@Value的区别

@Value注解是一个核心容器功能，它没有提供和type-safe配置属性相关的功能，下面这个表格总结了两者分别支持的功能：

| Feature                                                      | @ConfigurationProperties | @Value                   |
| :----------------------------------------------------------- | :----------------------- | :----------------------- |
| [宽松绑定](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-relaxed-binding) | Yes                      | Limited (see note below) |
| [元数据支持](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-configuration-metadata.html#configuration-metadata) | Yes                      | No                       |
| SpEL 表达式                                                  | No                       | Yes                      |

> 官方建议：
>
> - 如果你为自己的组件定义了一套配置，建议使用@ConfigurationProperties和POJO绑定，这样做能够提供结构化且类型安全的对象。
> - 如果硬要使用@Value，建议使用kebab-case形式，如@Value(" ${demo.item-price}")

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [SpringBoot官方文档：Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config)
- [Spring Boot干货系列：（二）配置文件解析](http://tengj.top/2017/02/28/springboot2/)
- [是时候彻底搞清楚 Spring Boot 的配置文件 application.properties 了！](http://www.javaboy.org/2019/0530/application.properties.html)
- [SpringBoot官方文档：Customize the Environment or ApplicationContext Before It Starts](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto-customize-the-environment-or-application-context)