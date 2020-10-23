[TOC]

# 一、快速开始

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
```

3. 编写MybatisPlus的配置

```java
@Configuration
@MapperScan("com.hyh.eblog.mapper")
public class MybatisPlusConfig {
}
```

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

2. 添加模板引擎，我们之前已经引入过

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

# 三、分页插件使用

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

# 四、条件构造器

地址: [条件构造器](https://mybatis.plus/guide/wrapper.html#abstractwrapper)

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