[TOC]

# 零、MybatisPlus特性：

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
@TableName("user") //指定表名 也可以使用table-prefix
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

代码样例地址：[spring-boot-mybatis-plus学习](https://gitee.com/tqbx/springboot-samples-learn/tree/master/spring-boot-mybatis-plus)

# 五、分页插件使用

文档地址：[分页插件使用](https://mybatis.plus/guide/interceptor.html)

我们这里采用SpringBoot的注解方式使用插件：

```java
@Configuration
@MapperScan("com.hyh.mybatisplus.mapper")
public class MybatisPlusConfig {

    /**
     * 新的分页插件,一缓和二缓遵循mybatis的规则,需要设置 MybatisConfiguration#useDeprecatedExecutor = false 避免缓存出现问题(该属性会在旧插件移除后一同移除)
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }

    @Bean
    public ConfigurationCustomizer configurationCustomizer() {
        return configuration -> configuration.setUseDeprecatedExecutor(false);
    }
}
```

测试分页：

```java
    @Test
    public void selectPage(){
        QueryWrapper<User> queryWrapper = new QueryWrapper<>();
        queryWrapper.ge("age",26);
        //Page<User> page = new Page<>(1,2);
        Page<User> page = new Page<>(1,2,false);
        Page<User> p = mapper.selectPage(page, queryWrapper);
        System.out.println("总页数: " + p.getPages());
        System.out.println("总记录数: " + p.getTotal());
        List<User> records = p.getRecords();
        records.forEach(System.out::println);
    }
```



# 七、MyBatisPlus代码生成器整合

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

# 八、ActiveRecord模式

1. 实体类需要继承Model类。
2. mapper继承BaseMapper。
3. 可以将实体类作为方法的调用者。

```java

    @Test
    public void selectById(){
        User user = new User();
        User selectById = user.selectById(1319899114158284802L);//新对象
        System.out.println(selectById == user); //false
        System.out.println(selectById);
    }
```

# 九、主键策略

博客地址：[MybatisPlus的各种支持的主键策略！](https://www.cnblogs.com/summerday152/p/13870612.html)

# 十、MybatisPlus的配置

文档地址：https://mybatis.plus/config/#%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE

Springboot的使用方式

```java
mybatis-plus:
  ......
  configuration:
    ......
  global-config:
    ......
    db-config:
      ......  
```

## mapperLocations

- 类型：`String[]`
- 默认值：`["classpath*:/mapper/**/*.xml"]`

MyBatis Mapper 所对应的 XML 文件位置，如果您在 Mapper 中有自定义方法(XML 中有自定义实现)，需要进行该配置，告诉 Mapper 所对应的 XML 文件位置。

Maven 多模块项目的扫描路径需以 `classpath*:` 开头 （即加载多个 jar 包下的 XML 文件）

# 十一、通用Service

```java
/**
 * Service接口,继承IService
 * @author Summerday
 */
public interface UserService extends IService<User> {
}

/**
 * Service实现类,继承ServiceImpl,实现接口
 * @author Summerday
 */
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {

}

@RunWith(SpringRunner.class)
@SpringBootTest
public class ServiceTest {

    @Autowired
    UserService userService;

    /**
     * 链式查询
     */
    @Test
    public void chain() {
        List<User> users = userService
                .lambdaQuery()
                .gt(User::getAge, 25)
                .like(User::getName, "雨")
                .list();
        for (User user : users) {
            System.out.println(user);
        }
    }

}
```

# 逻辑删除功能

只对自动注入的sql起效:

- 插入: 不作限制
- 查找: 追加where条件过滤掉已删除数据,且使用 wrapper.entity 生成的where条件会忽略该字段
- 更新: 追加where条件防止更新到已删除数据,且使用 wrapper.entity 生成的where条件会忽略该字段
- 删除: 转变为 更新

注：

- 逻辑删除是为了方便数据恢复和保护数据本身价值等等的一种方案，但实际就是删除。

- 如果你需要频繁查出来看就不应使用逻辑删除，而是以一个状态去表示。

## 逻辑删除的配置

- 步骤一：配置yml，全局配置，3.3.0版本之后，配置`logic-delete-field`后可以忽略不配置步骤2。

```yml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: deleted  # 全局逻辑删除的实体字段名(since 3.3.0,配置后可以忽略不配置步骤2)
      logic-delete-value: 1 # 逻辑已删除值(默认为 1)
      logic-not-delete-value: 0 # 逻辑未删除值(默认为 0)
```

- 步骤二：加上逻辑删除的注解，局部配置。

```java
    // 逻辑删除字段
    @TableLogic
    private Integer deleted;
```

## 逻辑删除的测试

```java
@Test
void logicDelete(){
    //UPDATE user SET deleted=1 WHERE deleted=0 AND (name = ?)
    boolean remove = userService.remove(
        new QueryWrapper<User>().eq("name", "summer"));
    System.out.println(remove);

}
@Test
void select(){
    //SELECT id,name,deleted FROM user WHERE deleted=0
    List<User> list = userService.list(
        new QueryWrapper<User>().select("id","name","deleted"));
    System.out.println(list);
}
@Test
public void updateById(){
    //UPDATE user SET age=? WHERE id=? AND deleted=0
    User user = new User();
    user.setId(1320008348799713282L);
    user.setAge(20);
    boolean b = userService.updateById(user);
    System.out.println(b);
}
```

## 查询中排除标识字段

使用注解标识需要排除的字段：

```java
    @TableField(select = false)
    private Integer deleted;
```

# 自动填充

- 在指定字段标注注解，生成器策略部分也可以配置。

```java
    // 创建时间
    @TableField(fill = FieldFill.INSERT)
    private Date createTime;

    // 更新时间
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Date updateTime;
```

```java
public enum FieldFill {
    /**
     * 默认不处理
     */
    DEFAULT,
    /**
     * 插入填充字段
     */
    INSERT,
    /**
     * 更新填充字段
     */
    UPDATE,
    /**
     * 插入和更新填充字段
     */
    INSERT_UPDATE
}
```

- 实现元对象处理接口：`com.baomidou.mybatisplus.core.handlers.MetaObjectHandler`

```java

@Slf4j
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    // fieldName 指的是实体类的属性名,而不是数据库的字段名
    @Override
    public void insertFill(MetaObject metaObject) {
        log.info("start insert fill ....");
        this.strictInsertFill(
            metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(
            metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        log.info("start update fill ....");
        this.strictUpdateFill(
            // 起始版本 3.3.0(推荐)
            metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        // 或者
        this.strictUpdateFill(
            // 起始版本 3.3.3(推荐)
            metaObject, "updateTime", LocalDateTime::now, LocalDateTime.class); 
    }
}
```

# 乐观锁插件

> 乐观锁适用于**读多写少**的场景。

乐观锁的实现机制：

1. 取出记录时，获取当前version
2. 更新时，带上这个version
3. 执行更新时， set version = newVersion where version = oldVersion
4. 如果version不对，就更新失败

使用方法：

- 在字段上加上@Version注解。

```java
    // 版本号
    @Version
    private Integer version;
```

> - **支持的数据类型只有:int,Integer,long,Long,Date,Timestamp,LocalDateTime**
> - 整数类型下 `newVersion = oldVersion + 1`
> - `newVersion` 会回写到 `entity` 中
> - 仅支持 `updateById(id)` 与 `update(entity, wrapper)` 方法
> - **在 `update(entity, wrapper)` 方法下, `wrapper` 不能复用!!!**

- 配置乐观锁插件

```java
    @Bean
    public OptimisticLockerInterceptor optimisticLockerInterceptor() {
        return new OptimisticLockerInterceptor();
    }
```

- 测试，为更新的实体设置期望的版本号：

```java
    @Test
    void update() {
        //PDATE user SET name=?, update_time=?, version=? WHERE id=? 
        // AND version=? AND deleted=0
        int version = 2;
        User user = new User();
        user.setId(1320037517763842049L);
        user.setName("sm2");
        user.setVersion(version);//期望的版本号
        boolean b = userService.updateById(user);
        System.out.println(b);
    }
```

# 六、SQL分析打印

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