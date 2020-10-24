[TOC]

# 一、MybatisPlus特性：

- 无侵入，损耗小，强大的CRUD操作。
- 支持Lambda形式调用，支持多种数据库。
- 支持主键自动生成，支持ActiveRecord模式。
- 支持自定义全局通用操作，支持关键词自动转义。
- 内置代码生成器，分页插件，性能分析插件。
- 内置全局拦截插件，内置sql注入剥离器。

# 一、快速开始

前置准备：本文所有代码样例包括数据库脚本均已上传至码云：[spring-boot-mybatis-plus学习](https://gitee.com/tqbx/springboot-samples-learn/blob/master/spring-boot-mybatis-plus/src/test/java/com/hyh/mybatisplus/SpringBootMybatisPlusApplicationTests.java)

1. 引入依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.0</version>
</dependency>
```

2. 配置yml

```yml
spring:
  # mysql数据库连接
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:p6spy:mysql://localhost:3306/eblog?serverTimezone=GMT%2B8
    username: xxx
    password: xxx
    
# mybatis配置
mybatis-plus:
  mapper-locations:
    - classpath*:mapper/*.xml
# logging
logging:
  level:
    root: warn
    com.hyh.mybatisplus.mapper: trace
  pattern:
    console: '%p%m%n'
```

3. 编写MybatisPlus的配置

```java
@Configuration
@MapperScan("com.hyh.mybatisplus.mapper")
public class MybatisPlusConfig {
}
```

4. 编写测试方法

```java
    @Test
    void selectById() {
        User user = mapper.selectById(1087982257332887553L);
        System.out.println(user);
    }
```

# 二、常用注解

当遇到不可抗拒因素导致数据库表与实体表名或字段名不对应时，可以使用注解进行指定。

```java
@Data
@ToString
@TableName("user") //指定表名
public class User {

    @TableId //指定主键
    private Long id;

    @TableField("name") //指定字段名
    private String name;
    private String email;
    private Integer age;

    private Long managerId;

    @TableField("create_time")
    private Date createTime;

    @TableField(exist = false)//备注[数据库中无对应的字段]
    private String remark;
}
```

# 三、排除非表字段的三种方式

假设实体中存在字段且该字段只是临时为了存储某些数据，数据库表中并没有，此时有三种方法可以排除这类字段。

1. 使用transient修饰字段，此时字段无法进行序列化，有时会不符合需求。
2. 使用static修饰字段，此时字段就归属于类了，有时不符合需求。

以上两种方式在某种意义下，并不能完美解决这个问题，为此，mp提供了下面这种方式：

```java
    @TableField(exist = false)//备注[数据库中无对应的字段]
    private String remark;
```

# 四、MybatisPlus的查询

博客地址：[https://www.cnblogs.com/summerday152/p/13869233.html](https://www.cnblogs.com/summerday152/p/13869233.html)

代码样例地址：[spring-boot-mybatis-plus学习](https://gitee.com/tqbx/springboot-samples-learn/blob/master/spring-boot-mybatis-plus/src/test/java/com/hyh/mybatisplus/SpringBootMybatisPlusApplicationTests.java)

# 五、分页插件使用

文档地址：[分页插件使用](https://mybatis.plus/guide/page.html)

我们这里采用SpringBoot的注解方式使用插件：

```java
//Spring boot方式
@Configuration
@MapperScan("com.baomidou.cloud.service.*.mapper*")
public class MybatisPlusConfig {

    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor paginationInterceptor = new PaginationInterceptor();
        // 设置请求的页面大于最大页后操作， true调回到首页，false 继续请求  默认false
        // paginationInterceptor.setOverflow(false);
        // 设置最大单页限制数量，默认 500 条，-1 不受限制
        // paginationInterceptor.setLimit(500);
        // 开启 count 的 join 优化,只针对部分 left join
        paginationInterceptor.setCountSqlParser(new JsqlParserCountOptimize(true));
        return paginationInterceptor;
    }
}
```



# 五、SQL分析打印

地址: [执行 SQL 分析打印](https://mybatis.plus/guide/p6spy.html)

1. 引入maven依赖

```xml
<dependency>
  <groupId>p6spy</groupId>
  <artifactId>p6spy</artifactId>
  <version>最新版本</version>
</dependency>
```

2. 配置application.yml

```yml
spring:
  datasource:
    driver-class-name: com.p6spy.engine.spy.P6SpyDriver
    url: jdbc:p6spy://localhost:3306/eblog?serverTimezone=GMT%2B8
    ...
```

3. spy.properties配置

```properties
#3.2.1以上使用
modulelist=com.baomidou.mybatisplus.extension.p6spy.MybatisPlusLogFactory,com.p6spy.engine.outage.P6OutageFactory
#3.2.1以下使用或者不配置
#modulelist=com.p6spy.engine.logging.P6LogFactory,com.p6spy.engine.outage.P6OutageFactory
# 自定义日志打印
logMessageFormat=com.baomidou.mybatisplus.extension.p6spy.P6SpyLogger
#日志输出到控制台
appender=com.baomidou.mybatisplus.extension.p6spy.StdoutLogger
# 使用日志系统记录 sql
#appender=com.p6spy.engine.spy.appender.Slf4JLogger
# 设置 p6spy driver 代理
deregisterdrivers=true
# 取消JDBC URL前缀
useprefix=true
# 配置记录 Log 例外,可去掉的结果集有error,info,batch,debug,statement,commit,rollback,result,resultset.
excludecategories=info,debug,result,commit,resultset
# 日期格式
dateformat=yyyy-MM-dd HH:mm:ss
# 实际驱动可多个
#driverlist=org.h2.Driver
# 是否开启慢SQL记录
outagedetection=true
# 慢SQL记录标准 2 秒
outagedetectioninterval=2
```

> 如何在控制台打印sql语句的执行结果？

```sql
<==    Columns: id, content, authorAvatar
<==        Row: 2, <<BLOB>>, /res/images/avatar/0.jpg
<==        Row: 1, <<BLOB>>, /res/images/avatar/default.png
<==      Total: 2
```

配置`mybatis-plus.configuration.log-impl`为`org.apache.ibatis.logging.stdout.StdOutImpl`

# 二、MyBatisPlus代码生成器整合

文档地址：[MyBatisPlus代码生成器整合](https://mybatis.plus/guide/generator.html#%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B)

MyBatis-Plus 从 `3.0.3` 之后移除了代码生成器与模板引擎的默认依赖，需要手动添加相关依赖：

1. 添加代码生成器依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.4.0</version>
</dependency>
```

