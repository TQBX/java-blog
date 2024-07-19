[toc]

## 一、SpringMVC简介

> 参考于：[https://www.cnblogs.com/myitnews/p/11565941.html#autoid-1-0-0](https://www.cnblogs.com/myitnews/p/11565941.html#autoid-1-0-0)
>
> Spring Web MVC是一种基于Java的实现了Web MVC设计模式的**请求驱动类型的轻量级Web框架**，即使用了**MVC架构模式**的思想，将web层进行职责解耦，基于请求驱动指的就是使用**请求-响应**模型，框架的目的就是帮助我们**简化开发**，Spring Web MVC也是要简化我们日常Web开发的。与之相反的是基于组件的、事件驱动的Web框架，如Tapestry、JSF等。
>
> - 前端控制器是DispatcherServlet；
> - 应用控制器其实拆为处理器映射器(Handler Mapping)进行处理器管理和视图解析器(View Resolver)进行视图管理；
> - 页面控制器/动作/处理器为Controller接口（仅包含ModelAndView handleRequest(request, response) 方法）的实现（也可以是任何的POJO类）；
> - 支持本地化（Locale）解析、主题（Theme）解析及文件上传等；
> - 提供了非常灵活的数据验证、格式化和数据绑定机制；
> - 提供了强大的约定大于配置（惯例优先原则）的契约式编程支持。

## 二、Spring的MVC运行流程

![](img/SpringMVC%E6%B5%81%E7%A8%8B/mvc.png)

1. 用户请求发送给DispatcherServlet，DispatcherServlet调用HandlerMapping处理器映射器；

2. HandlerMapping根据xml或注解找到对应的处理器，生成处理器对象【其实返回的是一个执行器链：包含handler和多个拦截器Interceptor】返回给DispatcherServlet；
3. DispatcherServlet会调用相应的HandlerAdapter；
4. HandlerAdapter经过适配调用具体的处理器去处理请求，生成ModelAndView返回给DispatcherServlet
5. DispatcherServlet将ModelAndView传给ViewReslover解析生成View返回给DispatcherServlet；
6. DispatcherServlet根据View进行渲染视图，最后响应给用户。

## 三、SpringMVC常用注解

### @Controller

- 标注控制层组件，标记的类就是一个SpringMVC Controller对象。
- 分发处理器将会扫描使用了该注解的类的方法，并检测方法是否使用使用@RequestMapping注解。
- 可以把request请求header部分的值绑定到方法参数上。
- 在标注的方法上，试图解析器可以解析jsp、html等页面，若要返回json，需要加上@ResponseBody 。

类似的表示组件作用的还有：

- @Service：表示业务层组件。
- @Repository：表示持久层组件。
- @Component：一般组件。

### @RestController

- 等于@Controller + @ResponseBody 
- 直接返回json，试图解析器无法再解析html等页面。
- 如果想返回页面，可以返回一个ModelAndView对象：`return new ModelAndView("index")`。

```java
@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello(){
        return "this is my spring-boot-quick-start...";
    }

    @GetMapping("go")
    public ModelAndView go(){
        ModelAndView mv = new ModelAndView("index");
        return mv;
    }
}
```

### @ControllerAdvice

- 用于全局异常处理，配合`ExceptionHandler`，捕获Controller中抛出指定类型的异常。

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ResponseBody
    @ExceptionHandler(value = ApiException.class)
    public AjaxResult handle(ApiException e){
        if(e.getErrorCode()!=null){
            return AjaxResult.error(e.getErrorCode().getCode(), e.getMessage());
        }
        return AjaxResult.error(e.getMessag());
    }
}
```

- 全局数据绑定，配合方法注解@InitBinder，用于request中自定义参数解析方式进行注册，达到自定义指定 格式参数的目的。

```java
@ControllerAdvice
public class MyControllerAdvice {

    @InitBinder
    public void globalInitBinder(WebDataBinder binder){
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }
}
```

-  全局数据预处理，结合方法型注解@ModelAttribute，表示其标注的方法将会在目标Controller方法执行之前执行。

```java
@ControllerAdvice
public class MyControllerAdvice {

    @ModelAttribute(value = "message")//在所有拦截器的preHandler方法执行之后执行
    public String globalModelAttribute(){
        System.out.println("MyControllerAdvice.globalModelAttribute");
        return "test";
    }
}

@RestController
public class HelloController {

    @GetMapping("go")
    public ModelAndView go(@ModelAttribute("message") String message){
        System.out.println(message); //"test"
        ModelAndView mv = new ModelAndView(message);
        return mv; //跳转到 test.html页面
    }
}
```

###  @RequestBody

- 作用于形参列表，将前台传来的数据【xml或json】封装为javabean。通常用于接收post请求的请求体。

### @ResponseBody

- 作用于方法上，通过适当的HttpMessageConverter将Controller方法返回的对象进行转换，写入Response body。
- 返回的数据是【xml或json】。

### @RequestParam

- 用于在SpringMVC后台控制层获取参数。
- 有三个参数：
  - value：请求参数名（必须配置）
  - required：是否必需，默认为 true，即 请求中必须包含该参数，如果没有包含，将会抛出异常（可选配置）
  - defaultValue：默认值，如果设置了该值，required 将自动设为 false，无论你是否配置了required，配置了什么值，都是 false（可选配置）

### @RequestHeader

- 可以把Request请求header部分的值绑定到方法的参数上。

### @PathVariable

- 用于将请求URL中的模板变量映射到功能处理方法的参数上。

```java
    @GetMapping("/go/{id}")
    public String PathVa(@PathVariable("id") Long id){
        System.out.println(id);
        return "test";
    }
```

### @RequestMapping

- 处理请求地址映射的注解，可用于类或方法上。
- 用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径。

- 参数：
  - value：指定请求的实际地址，指定的地址可以是URI Template 模式；
  - method： 指定请求的method类型， GET、POST、PUT、DELETE等，在RequestMethod定义，同GetMapping等。`@RequestMapping(value = "/go",method = RequestMethod.GET)` = `GetMapping("/go")`
  - consumes： 指定处理请求的提交内容类型（Content-Type），例如`application/json, text/html;`
  - produces: 指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回；
  - params： 指定request中必须包含某些参数值时，才让该方法处理；
  - headers： 指定request中必须包含某些指定的header值，才能让该方法处理请求；

### @Autowired

- 可以对类成员变量、方法一级构造参数进行标注，完成自动装配。

- 首先将默认类型匹配的bean自动装配到属性中。
- 如果类型匹配的bean不止一个，接着根据名称匹配。
- 如果查询结果为空，则抛出异常，如果想避免，可以使用`required = false`。



## 四、参考资料

- [https://www.cnblogs.com/fnlingnzb-learner/p/9723834.html](https://www.cnblogs.com/fnlingnzb-learner/p/9723834.html)

- [https://www.cnblogs.com/myitnews/p/11567719.html](https://www.cnblogs.com/myitnews/p/11567719.html)

- [https://blog.csdn.net/qq_36761831/article/details/89053314](https://blog.csdn.net/qq_36761831/article/details/89053314)