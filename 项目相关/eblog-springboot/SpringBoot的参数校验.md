# SpringBoot的参数校验

## 本篇要点



## 后端参数校验的必要性

> **Validating data** is a common task that occurs throughout all application layers, from the presentation to the persistence layer. **Often the same validation** logic is implemented in each layer which is time consuming and error-prone.

在开发中，从表现层到持久化层，数据校验都是一项逻辑差不多，但容易出错的任务。前端框架往往会采取一些检查参数的手段，比如校验并提示信息，那么，既然前端已经存在校验手段，后端的校验是否还有必要，是否多余了呢？

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

## 使用Validator的案例

### 什么是Validator





## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [javax.validation.constraints](https://docs.jboss.org/hibernate/beanvalidation/spec/2.0/api/javax/validation/constraints/package-summary.html)

- [SpringFramework：JavaBean Validation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#validation-beanvalidation)
- [SpringBoot官方：Validation](https://docs.spring.io/spring-boot/docs/2.4.0/reference/html/spring-boot-features.html#boot-features-validation)
- [SpringBoot写后端接口，看这一篇就够了！](https://segmentfault.com/a/1190000024467109)

- [SpringBoot如何优雅的校验参数](https://segmentfault.com/a/1190000021473727)
- [Spring Boot 2.x基础教程：JSR-303实现请求参数校验](http://blog.didispace.com/spring-boot-learning-21-2-3/)