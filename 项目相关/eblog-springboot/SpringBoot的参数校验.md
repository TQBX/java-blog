[toc]

# SpringBoot的参数校验

## 本篇要点

> JDK1.8、SpringBoot2.3.4release

- 说明后端参数校验的必要性。
- 介绍**如何使用validator进行参数校验**。
- 介绍@Valid和@Validated的区别。
- 介绍**如何自定义约束注解**。
- 关于Bean Validation的前世今生，建议阅读文章：[ 不吹不擂，第一篇就能提升你对Bean Validation数据校验的认知](https://www.yourbatman.cn/x2y/55d56c0b.html)，介绍十分详细。

## 后端参数校验的必要性

在开发中，从表现层到持久化层，数据校验都是一项逻辑差不多，但容易出错的任务，

前端框架往往会采取一些检查参数的手段，比如校验并提示信息，那么，既然前端已经存在校验手段，后端的校验是否还有必要，是否多余了呢？

并不是，正常情况下，参数确实会经过前端校验传向后端，但**如果后端不做校验，一旦通过特殊手段越过前端的检测，系统就会出现安全漏洞。**

## 不使用Validator的参数处理逻辑

既然是参数校验，很简单呀，用几个`if/else`直接搞定：

```java
    @PostMapping("/form")
    public String form(@RequestBody Person person) {
        if (person.getName() == null) {
            return "姓名不能为null";
        }
        if (person.getName().length() < 6 || person.getName().length() > 12) {
            return "姓名长度必须在6 - 12之间";
        }
        if (person.getAge() == null) {
            return "年龄不能为null";
        }
        if (person.getAge() < 20) {
            return "年龄最小需要20";
        }
        // service ..
        return "注册成功!";
    }
```

写法干脆，但`if/else`太多，过于臃肿，更何况这只是区区一个接口的两个参数而已，要是需要更多参数校验，甚至更多方法都需要这要的校验，这代码量可想而知。于是，这种做法显然是不可取的，我们可以利用下面这种更加优雅的参数处理方式。

## Validator框架提供的便利

> **Validating data** is a common task that occurs throughout all application layers, from the presentation to the persistence layer. **Often the same validation** logic is implemented in each layer which is time consuming and error-prone.

如果依照下图的架构，对每个层级都进行类似的校验，未免过于冗杂。

![application layers](img/SpringBoot%E7%9A%84%E5%8F%82%E6%95%B0%E6%A0%A1%E9%AA%8C/application-layers.png)

>  Jakarta Bean Validation 2.0 - **defines a metadata model and API for entity and method validation**. The default metadata source are **annotations**, with the ability to override and extend the meta-data through the use of XML. 
>
>  The API is not tied to a specific application tier nor programming model. It is specifically not tied to either web or persistence tier, and is available for both server-side application programming, as well as rich client Swing application developers.

`Jakarta Bean Validation2.0`定义了一个元数据模型，为实体和方法提供了数据验证的API，默认将注解作为源，可以通过XML扩展源。

![application layers2](img/SpringBoot%E7%9A%84%E5%8F%82%E6%95%B0%E6%A0%A1%E9%AA%8C/application-layers2.png)

## SpringBoot自动配置ValidationAutoConfiguration
`Hibernate Validator`是` Jakarta Bean Validation`的参考实现。

在SpringBoot中，只要类路径上存在JSR-303的实现，如`Hibernate Validator`，就会自动开启Bean Validation验证功能，这里我们只要引入`spring-boot-starter-validation`的依赖，就能完成所需。

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
```

目的其实是为了引入如下依赖：

```xml
    <!-- Unified EL 获取动态表达式-->
	<dependency>
      <groupId>org.glassfish</groupId>
      <artifactId>jakarta.el</artifactId>
      <version>3.0.3</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.hibernate.validator</groupId>
      <artifactId>hibernate-validator</artifactId>
      <version>6.1.5.Final</version>
      <scope>compile</scope>
    </dependency>
```

SpringBoot对BeanValidation的支持的自动装配定义在`org.springframework.boot.autoconfigure.validation.ValidationAutoConfiguration`类中，提供了默认的`LocalValidatorFactoryBean`和支持方法级别的拦截器`MethodValidationPostProcessor`。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(ExecutableValidator.class)
@ConditionalOnResource(resources = "classpath:META-INF/services/javax.validation.spi.ValidationProvider")
@Import(PrimaryDefaultValidatorPostProcessor.class)
public class ValidationAutoConfiguration {

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	@ConditionalOnMissingBean(Validator.class)
	public static LocalValidatorFactoryBean defaultValidator() {
        //ValidatorFactory
		LocalValidatorFactoryBean factoryBean = new LocalValidatorFactoryBean();
		MessageInterpolatorFactory interpolatorFactory = new MessageInterpolatorFactory();
		factoryBean.setMessageInterpolator(interpolatorFactory.getObject());
		return factoryBean;
	}

    // 支持Aop，MethodValidationInterceptor方法级别的拦截器
	@Bean
	@ConditionalOnMissingBean
	public static MethodValidationPostProcessor methodValidationPostProcessor(Environment environment,
			@Lazy Validator validator) {
		MethodValidationPostProcessor processor = new MethodValidationPostProcessor();
		boolean proxyTargetClass = environment.getProperty("spring.aop.proxy-target-class", Boolean.class, true);
		processor.setProxyTargetClass(proxyTargetClass);
        // factory.getValidator(); 通过factoryBean获取了Validator实例，并设置
		processor.setValidator(validator);
		return processor;
	}

}

```


## Validator+BindingResult优雅处理

> 默认已经引入相关依赖。

### 为实体类定义约束注解

```java
/**
 * 实体类字段加上javax.validation.constraints定义的注解
 * @author Summerday
 */

@Data
@ToString
public class Person {
    private Integer id;
    
    @NotNull
    @Size(min = 6,max = 12)
    private String name;

    @NotNull
    @Min(20)
    private Integer age;
}
```

### 使用@Valid或@Validated注解

@Valid和@Validated在Controller层做方法参数校验时功能相近，具体区别可以往后面看。

```java
@RestController
public class ValidateController {

    @PostMapping("/person")
    public Map<String, Object> validatePerson(@Validated @RequestBody Person person, BindingResult result) {
        Map<String, Object> map = new HashMap<>();
        // 如果有参数校验失败，会将错误信息封装成对象组装在BindingResult里
        if (result.hasErrors()) {
            List<String> res = new ArrayList<>();
            result.getFieldErrors().forEach(error -> {
                String field = error.getField();
                Object value = error.getRejectedValue();
                String msg = error.getDefaultMessage();
                res.add(String.format("错误字段 -> %s 错误值 -> %s 原因 -> %s", field, value, msg));
            });
            map.put("msg", res);
            return map;
        }
        map.put("msg", "success");
        System.out.println(person);
        return map;
    }
}
```

### 发送Post请求，伪造不合法数据

这里使用IDEA提供的HTTP Client工具发送请求。

```json
POST http://localhost:8081/person
Content-Type: application/json

{
  "name": "hyh",
  "age": 10
}
```

响应信息如下：

```json
HTTP/1.1 200 
Content-Type: application/json
Transfer-Encoding: chunked
Date: Sat, 14 Nov 2020 15:58:17 GMT
Keep-Alive: timeout=60
Connection: keep-alive

{
  "msg": [
    "错误字段 -> name 错误值 -> hyh 原因 -> 个数必须在6和12之间",
    "错误字段 -> age 错误值 -> 10 原因 -> 最小不能小于20"
  ]
}

Response code: 200; Time: 393ms; Content length: 92 bytes
```

## Validator + 全局异常处理

在接口方法中利用BindingResult处理校验数据过程中的信息是一个可行方案，但在接口众多的情况下，就显得有些冗余，我们可以利用全局异常处理，捕捉抛出的`MethodArgumentNotValidException`异常，并进行相应的处理。

### 定义全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * If the bean validation is failed, it will trigger a MethodArgumentNotValidException.
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Object> handleMethodArgumentNotValid(
        MethodArgumentNotValidException ex, HttpStatus status) {
        BindingResult result = ex.getBindingResult();
        Map<String, Object> map = new HashMap<>();
        List<String> list = new LinkedList<>();
        result.getFieldErrors().forEach(error -> {
            String field = error.getField();
            Object value = error.getRejectedValue();
            String msg = error.getDefaultMessage();
            list.add(String.format("错误字段 -> %s 错误值 -> %s 原因 -> %s", field, value, msg));
        });
        map.put("msg", list);
        return new ResponseEntity<>(map, status);
    }
}
```

### 定义接口

```java
@RestController
public class ValidateController {

    @PostMapping("/person")
    public Map<String, Object> validatePerson(@Valid @RequestBody Person person) {
        Map<String, Object> map = new HashMap<>();
        map.put("msg", "success");
        System.out.println(person);
        return map;
    }
}
```

## @Validated精确校验到参数字段

有时候，我们只想校验某个参数字段，并不想校验整个pojo对象，我们可以利用@Validated精确校验到某个字段。

### 定义接口

```java
@RestController
@Validated
public class OnlyParamsController {

    @GetMapping("/{id}/{name}")
    public String test(@PathVariable("id") @Min(1) Long id,
                       @PathVariable("name") @Size(min = 5, max = 10) String name) {
        return "success";
    }
}
```

### 发送GET请求，伪造不合法信息

```json
GET http://localhost:8081/0/hyh
Content-Type: application/json
```

未作任何处理，响应结果如下：

```json
{
  "timestamp": "2020-11-15T15:23:29.734+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "trace": "javax.validation.ConstraintViolationException: test.id: 最小不能小于1, test.name: 个数必须在5和10之间...省略",
  "message": "test.id: 最小不能小于1, test.name: 个数必须在5和10之间",
  "path": "/0/hyh"
}
```

可以看到，校验已经生效，但状态和响应错误信息不太正确，我们可以通过捕获`ConstraintViolationException`修改状态。

### 捕获异常，处理结果

```java
@ControllerAdvice
public class CustomGlobalExceptionHandler extends ResponseEntityExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(CustomGlobalExceptionHandler.class);


    /**
     * If the @Validated is failed, it will trigger a ConstraintViolationException
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public void constraintViolationException(ConstraintViolationException ex, HttpServletResponse response) throws IOException {
        ex.getConstraintViolations().forEach(x -> {
            String message = x.getMessage();
            Path propertyPath = x.getPropertyPath();
            Object invalidValue = x.getInvalidValue();
            log.error("错误字段 -> {} 错误值 -> {} 原因 -> {}", propertyPath, invalidValue, message);
        });
        response.sendError(HttpStatus.BAD_REQUEST.value());
    }
}
```

## @Validated和@Valid的不同

参考：[@Validated和@Valid的区别？教你使用它完成Controller参数校验（含级联属性校验）以及原理分析【享学Spring】](https://blog.csdn.net/f641385712/article/details/97621783)

- `@Valid`是标准JSR-303规范的标记型注解，用来标记验证属性和方法返回值，进行级联和递归校验。
- `@Validated`：是Spring提供的注解，是标准`JSR-303`的一个变种（补充），提供了一个分组功能，可以在入参验证时，根据不同的分组采用不同的验证机制。
- 在`Controller`中校验方法参数时，使用@Valid和@Validated并无特殊差异（若不需要分组校验的话）。
- `@Validated`注解可以用于类级别，用于支持Spring进行方法级别的参数校验。`@Valid`可以用在属性级别约束，用来表示**级联校验**。
- `@Validated`只能用在类、方法和参数上，而`@Valid`可用于方法、**字段、构造器**和参数上。

## 如何自定义注解

`Jakarta Bean Validation API`定义了一套标准约束注解，如@NotNull，@Size等，但是这些内置的约束注解难免会不能满足我们的需求，这时我们就可以自定义注解，创建自定义注解需要三步：

1. 创建一个constraint annotation。
2. 实现一个validator。
3. 定义一个default error message。

### 创建一个constraint annotation

```java
/**
 * 自定义注解
 * @author Summerday
 */

@Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE, TYPE_USE})
@Retention(RUNTIME)
@Constraint(validatedBy = CheckCaseValidator.class) //需要定义CheckCaseValidator
@Documented
@Repeatable(CheckCase.List.class)
public @interface CheckCase {
    String message() default "{CheckCase.message}";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    CaseMode value();

    @Target({FIELD, METHOD, PARAMETER, ANNOTATION_TYPE})
    @Retention(RUNTIME)
    @Documented
    @interface List {
        CheckCase[] value();
    }
}
```

### 实现一个validator

```java
/**
 * 实现ConstraintValidator
 *
 * @author Summerday
 */
public class CheckCaseValidator implements ConstraintValidator<CheckCase, String> {

    private CaseMode caseMode;

    /**
     * 初始化获取注解中的值
     */
    @Override
    public void initialize(CheckCase constraintAnnotation) {
        this.caseMode = constraintAnnotation.value();
    }

    /**
     * 校验
     */
    @Override
    public boolean isValid(String object, ConstraintValidatorContext constraintContext) {
        if (object == null) {
            return true;
        }

        boolean isValid;
        if (caseMode == CaseMode.UPPER) {
            isValid = object.equals(object.toUpperCase());
        } else {
            isValid = object.equals(object.toLowerCase());
        }

        if (!isValid) {
            // 如果定义了message值,就用定义的,没有则去
            // ValidationMessages.properties中找CheckCase.message的值
            if(constraintContext.getDefaultConstraintMessageTemplate().isEmpty()){
                constraintContext.disableDefaultConstraintViolation();
                constraintContext.buildConstraintViolationWithTemplate(
                        "{CheckCase.message}"
                ).addConstraintViolation();
            }
        }
        return isValid;
    }
}
```

### 定义一个default error message

在`ValidationMessages.properties`文件中定义：

```properties
CheckCase.message=Case mode must be {value}.
```

这样，自定义的注解就完成了，如果感兴趣可以自行测试一下，在某个字段上加上注解：`@CheckCase(value = CaseMode.UPPER)`。

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [javax.validation.constraints](https://docs.jboss.org/hibernate/beanvalidation/spec/2.0/api/javax/validation/constraints/package-summary.html)
- [SpringFramework：JavaBean Validation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#validation-beanvalidation)
- [SpringBoot官方：Validation](https://docs.spring.io/spring-boot/docs/2.4.0/reference/html/spring-boot-features.html#boot-features-validation)
- [SpringBoot写后端接口，看这一篇就够了！](https://segmentfault.com/a/1190000024467109)
- [SpringBoot如何优雅的校验参数](https://segmentfault.com/a/1190000021473727)
- [Spring Boot 2.x基础教程：JSR-303实现请求参数校验](http://blog.didispace.com/spring-boot-learning-21-2-3/)
- [ 不吹不擂，第一篇就能提升你对Bean Validation数据校验的认知](https://www.yourbatman.cn/x2y/55d56c0b.html)