[toc]

# SpringBoot中的全局异常处理

## 本篇要点

- 介绍SpringBoot默认的异常处理机制。
- 如何定义错误页面。
- 如何自定义异常数据。
- 介绍@ControllerAdvice注解处理异常。

## 一、SpringBoot默认的异常处理机制

默认情况下，SpringBoot为以下两种情况提供了不同的响应方式：

1. Browser Clients浏览器客户端：通常情况下请求头中的Accept会包含text/html，如果未定义/error的请求处理，就会出现如下html页面：Whitelabel Error Page，关于error页面的定制，接下来会详细介绍。

![](img/sb1.png)

2. Machine Clients机器客户端：Ajax请求，返回ResponseEntity实体json字符串信息。

```json
{
    "timestamp": "2020-10-30T15:01:17.353+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "trace": "java.lang.ArithmeticException: / by zero...",
    "message": "/ by zero",
    "path": "/"
}
```

SpringBoot默认提供了程序出错的结果映射路径/error，这个请求的处理逻辑在BasicErrorController中处理，处理逻辑如下：

```java
// 判断mediaType类型是否为text/html
@RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
    HttpStatus status = getStatus(request);
    Map<String, Object> model = Collections
        .unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
    response.setStatus(status.value());
    // 创建ModelAndView对象，返回页面
    ModelAndView modelAndView = resolveErrorView(request, response, status, model);
    return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
}

@RequestMapping
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
    HttpStatus status = getStatus(request);
    if (status == HttpStatus.NO_CONTENT) {
        return new ResponseEntity<>(status);
    }
    Map<String, Object> body = getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.ALL));
    return new ResponseEntity<>(body, status);
}
```

## 二、错误页面的定制

相信Whitelabel Error Pag页面我们经常会遇到，这样体验不是很好，在SpringBoot中可以尝试定制错误页面，定制方式主要分静态和动态两种：

1. **静态异常页面**

在`classpath:/public/error`或`classpath:/static/error`路径下定义相关页面：文件名应为**确切的状态代码**，如404.html，或**系列掩码**，如4xx.html。

举个例子，如果你想匹配404或5开头的所有状态代码到静态的html文件，你的文件目录应该是下面这个样子：

```java
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- public/
             +- error/
             |   +- 404.html
    		 |   +- 5xx.html
             +- <other public assets>
```

如果500.html和5xx.html同时生效，那么优先展示500.html页面。

2. **使用模板的动态页面**

放置在`classpath:/templates/error`路径下：这里使用thymeleaf模板举例：

```java
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- templates/
             +- error/
             |   +- 5xx.html
             +- <other templates>
```

页面如下：

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>5xx</title>
    </head>
    <body>
        <h1>5xx</h1>
        <table border="1">
            <tr>
                <td>path</td>
                <td th:text="${path}"></td>
            </tr>
            <tr>
                <td>error</td>
                <td th:text="${error}"></td>
            </tr>
            <tr>
                <td>message</td>
                <td th:text="${message}"></td>
            </tr>
            <tr>
                <td>timestamp</td>
                <td th:text="${timestamp}"></td>
            </tr>
            <tr>
                <td>status</td>
                <td th:text="${status}"></td>
            </tr>
        </table>
    </body>
</html>
```

> 如果静态页面和动态页面同时存在且都能匹配，SpringBoot对于错误页面的优先展示规则如下：
>
> 如果发生了500错误：
>
> 动态500 -> 静态500 -> 动态5xx -> 静态5xx

## 三、自定义异常数据

![](img/sb2.png)

## 参考阅读

- [Spring Boot干货系列：（十三）Spring Boot全局异常处理整理](http://tengj.top/2018/05/16/springboot13/#%E5%89%8D%E8%A8%80)


- [Springboot：Error-handling](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-error-handling)

- [江南一点雨： Spring Boot 中关于自定义异常处理的套路！](http://www.javaboy.org/2019/0417/springboot-exception.html)