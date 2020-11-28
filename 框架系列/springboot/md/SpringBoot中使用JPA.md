# SpringBoot中使用JPA

## 本篇要点

- 简单介绍JPA。

- 介绍快速SpringBoot快速整合JPA

## JPA是啥？

> The Java Persistence API is a standard technology that lets you “map” objects to relational databases. The spring-boot-starter-data-jpa POM provides a quick way to get started.

- JAP是`The Java Persistence API`标准，是一种能让对象能够快速映射到关系型数据库的技术。
- `spring-boot-starter-data-jpa`能够让你快速使用这门技术，它提供了以下依赖。
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

### 实体类

```java
@Entity
public class City implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String state;
	// 省略getter和setter
}
```

- @Entity标注保证实体能够被SpringBoot扫描到。

- @Id表明id。
- @GeneratedValue中标注主键生成策略。
- 如果对象名和表名不同，可以通过@Table指定。

### 启动项目，生成表

首先在数据库中创建jpa库，库名无所谓，和配置对应上就可以。

启动项目，你会发现控制台输出日志如下：

```java
Hibernate: drop table if exists city // 先删除
Hibernate: drop table if exists hibernate_sequence // 自增id策略，用来记录主键的表
Hibernate: create table city (id bigint not null, name varchar(255) not null, state varchar(255) not null, primary key (id)) engine=InnoDB // 创建表
Hibernate: create table hibernate_sequence (next_val bigint) engine=InnoDB
Hibernate: insert into hibernate_sequence values ( 1 )
```

此时我们配置的create效果已经显现，我们之后将它改为update，不然每次启动程序，数据表又得重建咯。

### 数据访问层

> [Working with Spring Data Repositories](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories)

Spring Data JPA repositories是你可以定义访问数据的接口，JPA查询是根据你的方法名称自动创建的。

对于更复杂的查询，您可以使用Spring Data的Query注解对方法进行注解。

这里我们编写一个接口，继承JpaRepository即可。City是对象名，不是表名，Integer为主键的类型。

```java
public interface CityDao extends JpaRepository<City,Integer> {
}
```

甚至不用在接口中写一个方法，它就已经拥有了各种增删改查的基本方法。



