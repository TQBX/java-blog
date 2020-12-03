# SpringBoot中使用JPA

## 本篇要点

- 简单介绍JPA。

- 介绍快速SpringBoot快速整合JPA

## JPA是啥？

> The Java Persistence API is a standard technology that lets you “map” objects to relational databases. The spring-boot-starter-data-jpa POM provides a quick way to get started.

- JPA是`The Java Persistence API`标准，Java持久层API，是一种能让对象能够快速映射到关系型数据库的技术规范。
- JPA只是一种规范，它需要第三方自行实现其功能，在众多框架中`Hibernate`是最为强大的一个。

## Spring Data JPA

`Spring Data JPA` 是采用基于JPA规范的`Hibernate`框架基础下提供了`Repository`层的实现。**`Spring Data Repository`极大地简化了实现各种持久层的数据库访问而写的样板代码量，同时`CrudRepository`提供了丰富的CRUD功能去管理实体类。**SpringBoot框架为Spring Data JPA提供了整合，`spring-boot-starter-data-jpa`能够让你快速使用这门技术，它提供了以下依赖。

- Hibernate：最流行的JPA实现之一。
- Spring Data JPA：帮助你去实现JPA-based repositories。
- Spring ORM：Spring Framework提供的核心ORM支持。

## 快速SpringBoot快速整合JPA

### 引入依赖

```xml
        <!--SpringBoot对jpa的封装-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <!--mysql驱动，8.x版本-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
```

### 配置yml

```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/jpa?serverTimezone=GMT%2B8
    username: root
    password: 123456
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    #在建表的时候，将默认的存储引擎切换为 InnoDB
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
    # 配置在日志中打印出执行的 SQL 语句信息。
    show-sql: true
    # 配置指明在程序启动的时候要删除并且创建实体类对应的表。
    hibernate:
      ddl-auto: create #update
```

值得注意的是：`spring.jpa.hibernate.ddl-auto`第一建表的时候可以create，指明在程序启动的时候要**删除并且创建实体类对应的表**。后续使用就需要改为update。

#### ddl-auto的几种属性值

- create：**每次加载hibernate时都会删除上一次的生成的表**，再重新根据model生成表，因此可能会导致数据丢失。
- create-drop ：每次加载hibernate时根据model类生成表，**但是sessionFactory一关闭，表就自动删除**。
- **update**：最常用的属性，**第一次加载hibernate时根据model类会自动建立起表**的结构（前提是先建立好数据库），以后加载hibernate时根据 model类**自动更新表结构**，原有数据不会清空，只会更新。
- validate ：每次加载hibernate时，会校验数据与数据库的字段类型是否相同，**字段不同会报错**。

### 实体类

JPA规范定义在`javax.persistence`包下，注意导包的时候不要导错。

```java
@Entity(name = "t_user")
@Data
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;
    private String password;


    @Transient
    private String email;

}
```

- @Entity标注保证实体能够被SpringBoot扫描到，对应表名为`t_user`。

- @Id表明id。

- @GeneratedValue中标注主键生成策略。
- @Transient表示不需要映射的字段。

#### 常见的主键生成策略

- **TABLE：** 使用一个特定的数据库表格来保存主键
- **SEQUENCE：** 根据底层数据库的序列来生成主键，条件是数据库支持序列。这个值要与generator一起使用，generator 指定生成主键使用的生成器（可能是orcale中自己编写的序列）。
- **IDENTITY：** 主键由数据库自动生成（主要是支持自动增长的数据库，如mysql）
- **AUTO：** 主键由程序控制，也是GenerationType的默认值。

### 启动项目，生成表

首先在数据库中创建jpa库，库名无所谓，和配置对应上就可以。

启动项目，你会发现控制台输出日志如下：

```java
Hibernate: drop table if exists t_user
Hibernate: create table t_user 
    (id bigint not null auto_increment, password varchar(255), username varchar(255), primary key (id)) engine=InnoDB
```

此时我们配置的create效果已经显现，我们之后将它改为update，不然每次启动程序，数据表又得重建咯。

### 数据访问层

> [Working with Spring Data Repositories](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories)

Spring Data JPA repositories是你可以定义访问数据的接口，JPA查询是根据你的方法名称自动创建的。

这里我们编写一个接口，继承JpaRepository即可。User是对象名，不是表名，Long为主键的类型。

```java
public interface UserDao extends JpaRepository<User, Long> {

    /**
     * 根据用户名和密码查询用户
     */
    User findByUsernameAndPassword(String username, String password);
}
```

JPA默认支持常见的增删改查，也支持`findByUsernameAndPassword`这种以字段命名的方法，对于更复杂的查询，您可以使用Spring Data的Query注解对方法进行注解。

#### 命名规范与对应SQL

![5-2](img/SpringBoot%E4%B8%AD%E4%BD%BF%E7%94%A8JPA/5-2-1606578228657.jpeg)

### 测试JPA

```java
@SpringBootTest
class SpringBootJpaApplicationTests {

    @Resource
    UserDao userDao;

    @Test
    void testJPA() {
        User user = userDao.save(new User(null, "summerday", "123456", "hangzhou"));
        System.out.println("添加用户: " + user);
        User u = userDao.findByUsernameAndPassword("summerday", "123456");
        System.out.println("根据用户名和密码查询用户: " + u);
        long count = userDao.count();
        System.out.println("当前用户数量: " + count);
        PageRequest page = PageRequest.of(0, 5, Sort.by(Sort.Order.desc("id")));
        Page<User> all = userDao.findAll(page);
        System.out.println("分页 + 根据id逆序 查询结果: " + all.getContent());
        if(userDao.existsById(u.getId())) {
            userDao.deleteById(u.getId());
            System.out.println("删除id为" + u.getId()+ "的用户成功");
        }
        long c = userDao.count();
        System.out.println("剩余用户数为: " + c);
    }
}
```

控制台输出如下：

![image-20201128221953880](img/SpringBoot%E4%B8%AD%E4%BD%BF%E7%94%A8JPA/image-20201128221953880.png)



## 五、源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [battcn：一起来学SpringBoot | 第六篇：整合SpringDataJpa](http://blog.battcn.com/2018/05/08/springboot/v2-orm-jpa/	)
- [方志朋：Spring Boot教程第4篇：JPA](https://www.fangzhipeng.com/springboot/2017/05/04/sb4-jpaJ.html)
- [江南一点雨：SpringBoot整合JPA](http://springboot.javaboy.org/2019/0407/springboot-jpa)