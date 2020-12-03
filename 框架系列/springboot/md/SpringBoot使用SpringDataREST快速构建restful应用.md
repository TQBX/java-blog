[toc]

## 本篇要点

- Spring Data REST的基本介绍。
- SpringBoot快速构建restful风格接口。

## Spring Data REST概述

REST Web服务已经成为Web上应用程序集成的第一大手段。 REST的核心是**定义一个包含与客户端进行交互资源的系统**。 这些资源以**超媒体驱动**的方式实现。 

Spring MVC和Spring WebFlux各自提供了构建REST服务的坚实基础。 但是，即使为`multi-domain`对象系统实现最简单的REST Web服务原则也可能很繁琐，并且会导致大量样板代码。

Spring Data REST旨在解决这个问题，它建立在Spring Data存储库之上，并自动将其导出为REST资源，客户端可以轻松查询并调用存储库本身暴露出来的接口。

## SpringBoot快速构建restful风格接口

SpringBoot构建`Spring Data REST`是相当方便的，因为自动化配置的存在，`spring-boot-starter-data-rest`可以让你不需要写多少代码，就能轻松构建一套完整的rest应用。

除此之外，你需要引入数据存储的依赖，它支持SpringData JPA、Spring Data MongoDB等，这里就使用JPA啦。正好我们前几篇介绍过JPA的简单使用：[SpringBoot整合Spring Data JPA](https://www.cnblogs.com/summerday152/p/14054637.html)

### 创建项目，导入依赖

可以选择在利用SpringInitializer创建的时候选择对应的依赖：

![image-20201201231908097](img/SpringBoot%E6%9E%84%E5%BB%BARestful%E9%A3%8E%E6%A0%BC%E6%8E%A5%E5%8F%A3/image-20201201231908097.png)

最终pom.xml中的依赖如下：

```xml
        <!--jpa-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <!--restful-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-rest</artifactId>
        </dependency>
        <!--web-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!--mysql-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
```

### yml配置

```yml
server:
  port: 8081
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/restful?serverTimezone=GMT%2B8
    username: root
    password: 123456
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  jpa:
    database: mysql
    #在建表的时候，将默认的存储引擎切换为 InnoDB
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
    # 配置在日志中打印出执行的 SQL 语句信息。
    show-sql: true
    # 配置指明在程序启动的时候要删除并且创建实体类对应的表。
    hibernate:
      ddl-auto: update
```

### 定义实体类

```java
@Entity(name = "t_user")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;
    private String password;

    @Column(length = 20)
    private Integer age;
}
```

### 定义Repository接口

```java
public interface UserDao extends JpaRepository<User, Long> {
}
```

到这里为止，其实可以发现，和JPA文章没啥差异，除了多引入了一个rest依赖。ok，启动项目，先把表生成了再说。

启动项目，我们就会发现JPA已经为我们将表结构创建完成，并且，**一个基于Restful风格的增删改查应用**也已诞生，我们可以使用接口测试工具，进行测试。

## 测试Restful接口

> 默认的请求路径是 类名首字母小写+后缀s，这里就是users。

### 测试添加功能

```json
POST: http://localhost:8081/users
{
    "username": "summerday",
    "password": "123456",
    "age": 18
}
```

添加成功之后，返回信息如下：

```json
{
  "username": "summerday",
  "password": "123456",
  "age": 18,
  "_links": {
    "self": {
      "href": "http://localhost:8081/users/1"
    },
    "user": {
      "href": "http://localhost:8081/users/1"
    }
  }
}
```

### 测试删除功能

```json
DELETE : http://localhost:8081/users/1
// 删除成功之后返回删除的 id = 1
```

### 测试修改功能

```json
PUT : http://localhost:8081/users/2
{
    "username": "summerday111",
    "password": "123456111",
    "age": 181
}
```

同样的，修改成功，将对应id为2的信息改为传入信息，并返回更新后的信息。

```json
{
  "username": "summerday111",
  "password": "123456111",
  "age": 181,
  "_links": {
    "self": {
      "href": "http://localhost:8081/users/2"
    },
    "user": {
      "href": "http://localhost:8081/users/2"
    }
  }
}
```

### 测试根据id查询

```json
GET : http://localhost:8081/users/3
```

### 测试分页查询

```json
GET : http://localhost:8081/users
```

分页查询结果：

```json
{
  "_embedded": {
    "users": [
      {
        "username": "summerday111",
        "password": "123456111",
        "age": 181,
        "_links": {
          "self": {
            "href": "http://localhost:8081/users/2"
          },
          "user": {
            "href": "http://localhost:8081/users/2"
          }
        }
      },
      {
        "username": "summerday",
        "password": "123456",
        "age": 18,
        "_links": {
          "self": {
            "href": "http://localhost:8081/users/3"
          },
          "user": {
            "href": "http://localhost:8081/users/3"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8081/users"
    },
    "profile": {
      "href": "http://localhost:8081/profile/users"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 2,
    "totalPages": 1,
    "number": 0
  }
}
```

### 测试分页+排序

```json
GET : http://localhost:8081/users?page=0&size=1&sort=age,desc
```

第一页，每页size为1的记录，按age逆序。

```json
{
  "_embedded": {
    "users": [
      {
        "username": "summerday111",
        "password": "123456111",
        "age": 181,
        "_links": {
          "self": {
            "href": "http://localhost:8081/users/2"
          },
          "user": {
            "href": "http://localhost:8081/users/2"
          }
        }
      }
    ]
  },
  "_links": {
    "first": {
      "href": "http://localhost:8081/users?page=0&size=1&sort=age,desc"
    },
    "self": {
      "href": "http://localhost:8081/users?page=0&size=1&sort=age,desc"
    },
    "next": {
      "href": "http://localhost:8081/users?page=1&size=1&sort=age,desc"
    },
    "last": {
      "href": "http://localhost:8081/users?page=1&size=1&sort=age,desc"
    },
    "profile": {
      "href": "http://localhost:8081/profile/users"
    }
  },
  "page": {
    "size": 1,
    "totalElements": 2,
    "totalPages": 2,
    "number": 0
  }
}
```

## 定制查询

### 自定义查询接口

```java
public interface UserDao extends JpaRepository<User, Long> {

    List<User> findUsersByUsernameContaining(@Param("username")String username);
}
```

访问：`http://localhost:8081/users/search`，查询自定义接口。

```json
{
  "_links": {
    "findUsersByUsernameContaining": {
      "href": "http://localhost:8081/users/search/findUsersByUsernameContaining{?username}",
      "templated": true
    },
    "self": {
      "href": "http://localhost:8081/users/search"
    }
  }
}
```

测试一下这个方法：

```json
GET : http://localhost:8081/users/search/findUsersByUsernameContaining?username=111

{
  "_embedded": {
    "users": [
      {
        "username": "summerday111",
        "password": "123456111",
        "age": 181,
        "_links": {
          "self": {
            "href": "http://localhost:8081/users/2"
          },
          "user": {
            "href": "http://localhost:8081/users/2"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:8081/users/search/findUsersByUsernameContaining?username=111"
    }
  }
}
```

### 自定义接口名

```java
public interface UserDao extends JpaRepository<User, Long> {
    //rel 表示接口查询中，这个方法的 key 
    //path 表示请求路径
    @RestResource(rel = "auth", path = "auth")
    User findByUsernameAndPassword(@Param("name") String username, @Param("pswd") String password);
}
```

继续查询自定义的接口：

```json
    "auth": {
      "href": "http://localhost:8081/users/search/auth{?name,pswd}",
      "templated": true
    }
```

测试一下：

```json
GET : http://localhost:8081/users/search/auth?name=summerday&pswd=123456
```

### 设置接口对前端隐藏

```java
    @Override
    @RestResource(exported = false)
    void deleteById(Long aLong);
```

### 定义生成JSON字符串的相关信息

```java
@RepositoryRestResource(collectionResourceRel = "userList",itemResourceRel = "u",path = "user")
public interface UserDao extends JpaRepository<User, Long> {
}
```

此时路径名为`/user`：

```json
GET : http://localhost:8081/user/2

{
  "username": "summerday111",
  "password": "123456111",
  "age": 181,
  "_links": {
    "self": {
      "href": "http://localhost:8081/user/2"
    },
    "u": {
      "href": "http://localhost:8081/user/2"
    }
  }
}
```

## 其他配置属性

Spring Data REST其他可配置的属性，通过`spring.data.rest.basePath=/v1`的形式指定。

| 属性                 | 描述                                                |
| -------------------- | --------------------------------------------------- |
| `basePath`           | the root URI                                        |
| `defaultPageSize`    | 每页默认size                                        |
| `maxPageSize`        | change the maximum number of items in a single page |
| `pageParamName`      | 配置分页查询时页码的 key，默认是 page               |
| `limitParamName`     | 配置分页查询时每页查询页数的 key，默认是size        |
| `sortParamName`      | 配置排序参数的 key ，默认是 sort                    |
| `defaultMediaType`   | 配置默认的媒体类型                                  |
| `returnBodyOnCreate` | 添加成功时是否返回添加记录                          |
| `returnBodyOnUpdate` | 更新成功时是否返回更新记录                          |

## 参考阅读

- [江南一点雨：Spring Boot 中 10 行代码构建 RESTful 风格应用](http://www.javaboy.org/2019/0606/springboot-restful.html)
- [官方文档：Spring-data-rest](https://docs.spring.io/spring-data/rest/docs/current/reference/html/#reference)