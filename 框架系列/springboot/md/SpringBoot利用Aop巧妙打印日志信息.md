[toc]

# SpringBoot利用AOP巧妙打印日志信息

## 本篇要点

- 简要回顾SpringAOP的相关知识点：关键术语，通知类型，切入点表达式等等。
- 介绍SpringBoot快速启动测试AOP，巧妙打印日志信息。

## 简单回顾SpringAOP的相关知识点

SpringAOP的相关的知识点包括源码解析，我已经在之前的文章中详细说明，如果对AOP的概念还不是特别清晰的话，推荐先阅读这篇文章：[SpringAOP+源码解析，切就完事了](https://www.cnblogs.com/summerday152/p/13652903.html)。

为了加深印象，这边再做一个简短的回顾：

### 1、AOP关键术语

![](https://img-blog.csdnimg.cn/2020050115000823.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NreV9RaWFvQmFfU3Vt,size_16,color_FFFFFF,t_70)

- **切面（Aspect）**：也就是我们定义的专注于提供辅助功能的模块，比如安全管理，日志信息等。
- **连接点（JoinPoint）**：切面代码可以通过连接点切入到正常业务之中，图中每个方法的每个点都是连接点。
- **切入点（PointCut）**：一个切面不需要通知所有的连接点，而**在连接点的基础之上增加切入的规则**，选择需要增强的点，最终真正通知的点就是切入点。
- **通知方法（Advice）**：就是切面需要执行的工作，主要有五种通知：before，after，afterReturning，afterThrowing，around。
- **织入（Weaving）**：将切面应用到目标对象并创建代理对象的过程，SpringAOP选择再目标对象的运行期动态创建代理对

- **引入（introduction）**：在不修改代码的前提下，引入可以在运行期为类动态地添加方法或字段。

### 2、通知的五种类型

- **前置通知Before**：目标方法调用之前执行的通知。
- **后置通知After**：目标方法完成之后，无论如何都会执行的通知。
- **返回通知AfterReturning**：目标方法成功之后调用的通知。
- **异常通知AfterThrowing**：目标方法抛出异常之后调用的通知。
- **环绕通知Around**：可以看作前面四种通知的综合。

### 3、切入点表达式

上面提到：连接点增加切入规则就相当于定义了切入点，当然切入点表达式分为很多种，这里主要学习execution和annotation表达式。

#### execution

- 写法：execution(访问修饰符 返回值 包名.包名……类名.方法名(参数列表))
- 例：`execution(public void com.smday.service.impl.AccountServiceImpl.saveAccount())`
- 访问修饰符可以省略，返回值可以使用通配符*匹配。
- 包名也可以使用`*`匹配，数量代表包的层级，当前包可以使用`..`标识，例如`* *..AccountServiceImpl.saveAccount()`
- 类名和方法名也都可以使用`*`匹配：`* *..*.*()`
- 参数列表使用`..`可以标识有无参数均可，且参数可为任意类型。

> 全通配写法：`* *…*.*(…)`

通常情况下，切入点应当设置再业务层实现类下的所有方法：`* com.smday.service.impl.*.*(..)`。

#### @annotation

**匹配连接点被它参数指定的Annotation注解的方法**。也就是说，所有被指定注解标注的方法都将匹配。

`@annotation(com.hyh.annotation.Log)`：指定Log注解方法的连接点。

### 4、AOP应用场景

- 记录日志
- 监控性能
- 权限控制
- 事务管理

## 快速开始

### 引入依赖

如果你使用的是SpringBoot，那么只需要引入：`spring-boot-starter-aop`，框架已经将`spring-aop`和`aspectjweaver`整合进去。

```java
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
```

### 定义日志信息封装

```java
/**
 * Controller层的日志封装
 * @author Summerday
 */
@Data
@ToString
public class WebLog implements Serializable {

    private static final long serialVersionUID = 1L;

    // 操作描述
    private String description;

    // 操作时间
    private Long startTime;

    // 消耗时间
    private Integer timeCost;

    // URL
    private String url;

    // URI
    private String uri;

    // 请求类型
    private String httpMethod;

    // IP地址
    private String ipAddress;

    // 请求参数
    private Object params;

    // 请求返回的结果
    private Object result;

    // 操作类型
    private String methodType;
}
```

### 自定义注解@Log

```java
@Target({ElementType.PARAMETER,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log {

    /**
     * 描述
     */
    String description() default "";

    /**
     * 方法类型 INSERT DELETE UPDATE OTHER
     */
    MethodType methodType() default MethodType.OTHER;
}
```

### 定义测试接口

```java
@RestController
public class HelloController {

    @PostMapping("/hello")
    @Log(description = "hello post",methodType = MethodType.INSERT)
    public String hello(@RequestBody User user) {
        return "hello";
    }

    @GetMapping("/hello")
    @Log(description = "hello get")
    public String hello(@RequestParam("name") String username, String hobby) {
        int a = 1 / 0;
        return "hello";
    }
}
```

### 定义切面Aspect与切点Pointcut

用@Aspect注解标注标识切面，用@PointCut定义切点。

```java
/**
 * 定义切面
 * @author Summerday
 */

@Aspect
@Component
public class LogAspect {

    private static final Logger log = LoggerFactory.getLogger(LogAspect.class);

    /**
     * web层切点
     * 1. @Pointcut("execution(public * com.hyh.web.*.*(..))")  web层的所有方法
     * 2. @Pointcut("@annotation(com.hyh.annotation.Log)")      Log注解标注的方法
     */
    @Pointcut("@annotation(com.hyh.annotation.Log)")
    public void webLog() {
    }
}
```

### 定义通知方法Advice

这里使用环绕通知，

```java
/**
 * 定义切面
 * @author Summerday
 */

@Aspect
@Component
public class LogAspect {

    private static final Logger log = LoggerFactory.getLogger(LogAspect.class);

    /**
     * web层切点
     * 1. @Pointcut("execution(public * com.hyh.web.*.*(..))")  web层的所有方法
     * 2. @Pointcut("@annotation(com.hyh.annotation.Log)")      Log注解标注的方法
     */

    @Pointcut("@annotation(com.hyh.annotation.Log)")
    public void webLog() {
    }


    /**
     * 环绕通知
     */
    @Around("webLog()")
    public Object doAround(ProceedingJoinPoint joinPoint) throws Throwable {
        //获取请求对象
        HttpServletRequest request = getRequest();
        WebLog webLog = new WebLog();
        Object result = null;
        try {
            log.info("=================前置通知=====================");
            long start = System.currentTimeMillis();
            result = joinPoint.proceed();
            log.info("=================返回通知=====================");
            long timeCost = System.currentTimeMillis() - start;
            // 获取Log注解
            Log logAnnotation = getAnnotation(joinPoint);
            // 封装webLog对象
            webLog.setMethodType(logAnnotation.methodType().name());
            webLog.setDescription(logAnnotation.description());
            webLog.setTimeCost((int) timeCost);
            webLog.setStartTime(start);
            webLog.setIpAddress(request.getRemoteAddr());
            webLog.setHttpMethod(request.getMethod());
            webLog.setParams(getParams(joinPoint));
            webLog.setResult(result);
            webLog.setUri(request.getRequestURI());
            webLog.setUrl(request.getRequestURL().toString());
            log.info("{}", JSONUtil.parse(webLog));
        } catch (Throwable e) {
            log.info("==================异常通知=====================");
            log.error(e.getMessage());
            throw new Throwable(e);
        }finally {
            log.info("=================后置通知=====================");
        }
        return result;
    }

    /**
     * 获取方法上的注解
     */
    private Log getAnnotation(ProceedingJoinPoint joinPoint) {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        return method.getAnnotation(Log.class);
    }


    /**
     * 获取参数 params:{"name":"天乔巴夏"}
     */
    private Object getParams(ProceedingJoinPoint joinPoint) {
        // 参数名
        String[] paramNames = getMethodSignature(joinPoint).getParameterNames();
        // 参数值
        Object[] paramValues = joinPoint.getArgs();
        // 存储参数
        Map<String, Object> params = new LinkedHashMap<>();
        for (int i = 0; i < paramNames.length; i++) {
            Object value = paramValues[i];
            // MultipartFile对象以文件名作为参数值
            if (value instanceof MultipartFile) {
                MultipartFile file = (MultipartFile) value;
                value = file.getOriginalFilename();
            }
            params.put(paramNames[i], value);
        }
        return params;
    }

    private MethodSignature getMethodSignature(ProceedingJoinPoint joinPoint) {
        return (MethodSignature) joinPoint.getSignature();
    }


    private HttpServletRequest getRequest() {
        ServletRequestAttributes requestAttributes =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        return requestAttributes.getRequest();
    }

}
```

这里处理webLog的方式有很多种，考虑性能，可以采用异步方式存入数据库，相应代码已经上传至Gitee。

### 测试

```java
POST http://localhost:8081/hello
Content-Type: application/json

{  "id" : 1,   "username" : "天乔巴夏",   "age": 18 }
```

结果如下：

```java
=================前置通知=====================
=================返回通知=====================
{"ipAddress":"127.0.0.1","description":"hello post","httpMethod":"POST","params":{"user":{"id":1,"age":18,"username":"天乔巴夏"}},"uri":"/hello","url":"http://localhost:8081/hello","result":"hello","methodType":"INSERT","startTime":1605596028383,"timeCost":28}
=================后置通知=====================
```

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [Spring Boot 使用AOP切面实现后台日志管理模块](https://blog.csdn.net/lchq1995/article/details/80663169)