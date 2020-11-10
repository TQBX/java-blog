[toc]

## 本篇要点

- 介绍如何一步步将SpringBoot项目部署到远程服务器上。

## 部署全流程

本文采用创建**可执行jar**的方式启动SpringBoot项目。

### 1、配置maven插件

```xml
    
    <packaging>jar</packaging> <!--打成jar包 -->
	<build>
        <!--打成jar包的名称-->
        <finalName>fireworks</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.3.5.RELEASE</version>
            </plugin>
            <plugin>
                <!--排除测试类对打包的干扰-->
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <testFailureIgnore>true</testFailureIgnore>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

### 2、mvn package或mvn install

> mvn package 和 mvn install的区别：
>
> 1. package将会编译代码，并打包，最终按照maven规定的packaging方式打包，最终输出到目标目录中。
> 2. install同样也会编译，并打包，但之后install还会将打好的包安装在本地仓库，供其他项目使用。

```shell
# 执行mvn clean，移除之前的target目录
mvn clean
# 切换到项目路径下，执行mvn package指令
mvn package

# 输出日志
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ fireworks ---
[INFO] Building jar: D:\Java_Project\firework2.0\target\fireworks.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:2.3.5.RELEASE:repackage (repackage) @ fireworks ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS

```

最终jar包输出到`D:\Java_Project\firework2.0\target\`目录下。进入该目录，会发现存在两个文件：`fireworks.jar`和`fireworks.jar.original`。

- `fireworks.jar`就是我们所说的可执行的jar，包含**已编译的类和运行需要的所有jar依赖**，如果你想看看里面有啥，可以通过如下命令：

```shell
$ jar tvf target/fireworks.jar
```

- `fireworks.jar.original`比前者小的多，这是**Maven在Spring Boot重新打包之前创建的原始jar文件**，通过上面的命令，可以看到里面没有运行需要的依赖，只包含我们定义的类编译后的.class文件。

因此，如果我们想要启动SpringBoot项目，需要使用可执行的jar，因为它具备所有的jar依赖，启动命令如下：

```shell
$ java -jar fireworks.jar
```

### 3、将jar包上传至远程服务器

这里使用winSCP，无论使用哪种工具，只要能够将文件上传到远程服务器上就可以。

### 4、在远程服务器上执行jar包

```shell
nohup java -jar fireworks-0.0.1-SNAPSHOT.jar & # 后台启动jar
```

注：如果之前启动过项目，记得将原先那个进程关闭：

```shell
ps -ef|grep java
kill -9 pid
```

## 参考阅读

- [如何将springboot项目打包成jar包并部署到服务器上](https://www.jianshu.com/p/f7c2d17cb17b)
- [【mvn打包】.jar 和 .jar.original的区别](https://www.cnblogs.com/ivan5277/articles/11917858.html)
- [SpringBoot：Creating an Executable Jar](https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started.html#getting-started-first-application)