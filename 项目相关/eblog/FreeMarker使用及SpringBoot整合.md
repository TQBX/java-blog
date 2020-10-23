[TOC]

详细文档地址：http://freemarker.foofun.cn/index.html

FreeMarker是什么？ 

- 一款模板引擎。即一种基于模板和要改变的数据， 并用来生成输出文本(HTML网页，电子邮件，配置文件，源代码等)的通用工具。

- 在模板中，你可以专注于如何展现数据， 而在模板之外可以专注于要展示什么数据，体现就是：模板+ 数据模型 = 输出。

![](http://freemarker.foofun.cn/figures/overview.png)

# 快速开始

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
4. 公共布局模板的定义,`<#nested >`标签定义个性的地方,引入时填充自定义代码

```html
<#-- 定义宏 -->
<#macro layout title>
    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="utf-8">
            <title>${title}</title>
        </head>
        <body>
            <#include "/inc/header.ftl"/><#--引入公共部分的内容-->
            <#nested > <#--此处内容可以自定义-->
            <#include "/inc/footer.ftl"/>
		<#--省略公共部分内容-->
</#macro>
```

5. 引入公共模板及插入自定义的代码

```html
<#include "/inc/layout.ftl"/><#-- 这里引入公共模板layout.ftl-->
<@layout "博客分类"> <#-- 这里对应<#macro layout title> 博客分类就是传入的title参数-->
	<#-- 这里定义特色内容 -->
</@layout>
```

# 模板一览

`${...}`： FreeMarker将会输出真实的值来替换大括号内的表达式，被称为插值表达式。

`<#../>`：FTL标签，以#开头，自定义标签则以@开头。

## 常用指令

### 条件指令：if、elseif、else

```html
<#if animals.python.price < animals.elephant.price>
  Pythons are cheaper than elephants today.
<#elseif animals.elephant.price < animals.python.price>
  Elephants are cheaper than pythons today.
<#else>
  Elephants and pythons cost the same today.
</#if>
```

### list指令

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

### include指令

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

## 内建函数