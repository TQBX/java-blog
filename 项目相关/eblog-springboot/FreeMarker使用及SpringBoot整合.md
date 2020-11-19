[TOC]

## 本篇要点

- 介绍FreeMark基本原理。
- 介绍SpringBoot与FreeMarker快速整合。

## FreeMarker是什么？ 

- 一款模板引擎。即一种基于模板和要改变的数据， 并用来生成输出文本(HTML网页，电子邮件，配置文件，源代码等)的通用工具。

- 在模板中，你可以专注于如何展现数据， 而在模板之外可以专注于要展示什么数据，体现就是：模板+ 数据模型 = 输出。

![img](img/FreeMarker%E4%BD%BF%E7%94%A8%E5%8F%8ASpringBoot%E6%95%B4%E5%90%88/overview.png)

## 快速开始

1. pom.xml确定导入FreeMarker依赖包

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-freemarker</artifactId>
    </dependency>
```

2. 配置application.yml相关配置

```yml
spring:
  freemarker:
    settings:
      classic_compatible: true #处理空值
      datetime_format: yyy-MM-dd HH:mm
      number_format: 0.##
    suffix: .ftl
    template-loader-path:
      - classpath:/templates
```

3. 在templates目录下放置`.ftl`文件，意为`freemarker templates layer`。
4. 编写Controller，将模型存入request中：

```java
@Controller
public class TestController{

    @GetMapping("test")
    public String test(HttpServletRequest request){
        List<User> users = new LinkedList<>();
        for(int i = 0; i < 10; i ++){
            User user = new User();
            user.setId((long)i);
            user.setUsername("name = " + i);
            users.add(user);
        }
        request.setAttribute("users",users);

        return "test";
    }
}
```

5. 简单使用FreeMarker的标签指令：

```html
<#list users>
    <p>Users:
    <ul>
        <#items as user>
            <li>${user.id}--${user.username}<br>
            </#items>
    </ul>
<#else>
    <p>no users!
</#list>
```

## 模板一览

`${...}`： FreeMarker将会输出真实的值来替换大括号内的表达式，被称为插值表达式。

`<#../>`：FTL标签，以#开头，自定义标签则以@开头。

### 常用指令

#### 条件指令：if、elseif、else

```html
<#if animals.python.price < animals.elephant.price>
  Pythons are cheaper than elephants today.
<#elseif animals.elephant.price < animals.python.price>
  Elephants are cheaper than pythons today.
<#else>
  Elephants and pythons cost the same today.
</#if>
```

#### list指令

`list` 指令的一般格式为： `<#list sequence as loopVariable> repeatThis`。 `repeatThis` 部分将会在给定的 `sequence` 遍历时在每一项中重复， 从第一项开始，一个接着一个。在所有的重复中， `loopVariable` 将持有当前遍历项的值。 这个变量仅存在于 `<#list ...>` 和 `<#list>` 标签内。

```html
<p>We have these animals:
<table border=1>
  <#list animals as animal>
    <tr><td>${animal.name}<td>${animal.price} Euros
  </#list>
</table>
```

`sequence` 可以是任意表达式， 比如我们可以列表显示示例数据模型中的水果，就像这样：

```html
<ul>
<#list misc.fruits as fruit>
  <li>${fruit}
</#list>
</ul>
```

上面示例中的一个问题是如果我们有0个水果，它仍然会输出一个空的 `<ul></ul>`，而不是什么都没有。 要避免这样的情况，可以这么来使用 `list`：

```html
<#list misc.fruits>
  <p>Fruits:
  <ul>
    <#items as fruit>
      <li>${fruit}<#sep> and</#sep>
    </#items>
  </ul>
<#else>
  <p>We have no fruits.
</#list>
```

#### include指令

使用 `include` 指令， 我们可以在模板中插入其他文件的内容。

如果需要在每个页面的下方都显示版权信息，可以将版权信息单独放在页面文件 `copyright_footer.html` 中：

```html
<hr>
<i>
Copyright (c) 2000 <a href="http://www.acmee.com">Acmee Inc</a>,
<br>
All Rights Reserved.
</i>
```

使用时，用include引入该文件即可：

```html
<html>
<head>
  <title>Test page</title>
</head>
<body>
  <h1>Test page</h1>
  <p>Blah blah...
  <#include "/copyright_footer.html">
</body>
</html>
```

### 内建函数

内建函数很像子变量(如果了解Java术语的话，也可以说像方法)， 它们并不是数据模型中的东西，是 FreeMarker 在数值上添加的。 为了清晰子变量是哪部分，使用 `?`(问号)代替 `.`(点)来访问它们。

所有内建函数参考：http://freemarker.foofun.cn/ref_builtins.html

### 处理不存在的变量

一个不存在的变量和一个是`null`值的变量， 对于FreeMarker来说是一样的。

不论在哪里引用变量，都可以指定一个默认值来避免变量丢失这种情况， 通过在变量名后面跟着一个 `!`(叹号，译者注)和默认值。 就像下面的这个例子，当 `user` 不存在于数据模型时, 模板将会将 `user` 的值表示为字符串 `"visitor"`。(当 `user` 存在时， 模板就会表现出 `${user}` 的值)：

```html
<h1>Welcome ${user!"visitor"}!</h1>
```

也可以在变量名后面通过放置 `??` 来询问一个变量是否存在。将它和 `if` 指令合并， 那么如果 `user` 变量不存在的话将会忽略整个问候的代码段：

```html
<#if user??><h1>Welcome ${user}!</h1></#if>
```

## 自定义指令

### 使用macro定义宏

- 未带参数宏调用

```html
<#macro greet>
  <font size="+2">Hello Joe!</font>
</#macro>

<#--未带参数宏调用-->
    <@greet></@greet>
```

- 带参数宏调用

```html
<#macro greet_2 person>
    <font size="+2">Hello ${person}!</font>
</#macro>
<#--带参数宏调用-->
    <@greet_2 person="天乔巴夏"/>
```

- 嵌套调用

```html
<#macro nest_test>
    <#nested >
</#macro>
<#--嵌套调用-->    
    <@nest_test>
        hyh
        </@nest_test>
```

### 使用TemplateDirectiveModel扩展。

这部分可以参考我发在码云上的代码：https://gitee.com/tqbx/springboot-samples-learn/tree/master/spring-boot-freemarker。

```java
    @Override
    public void execute(DirectiveHandler handler) throws Exception {
        // 获取参数
        String username = handler.getString("username");
        String city = handler.getString("city");
        // 处理参数
        String template = "{}来自{}.";
        String format = StrUtil.format(template, username, city);
        // 传回去
        handler.put("result",format).render();
    }

```

```java
@Configuration
public class FreeMarkerConfig {

    @Autowired
    StringTemplate stringTemplate;
    @Autowired
    private freemarker.template.Configuration configuration;


    @PostConstruct
    public void setUp() {
        configuration.setSharedVariable("timeAgo", new TimeAgoMethod());
        configuration.setSharedVariable("strstr", stringTemplate);
    }
}
```

```html
<@strstr username="hyh" city="杭州">
    ${result}
    </@strstr>
```

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：https://gitee.com/tqbx/springboot-samples-learn，另有其他SpringBoot的整合哦。

## 参考阅读

- http://freemarker.foofun.cn/index.html