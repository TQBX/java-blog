

[toc]

# SpringBoot瘦身部署

## 本片要点

- 介绍如何为jar包瘦身，方便部署。

## 正常打包部署的方式

之前已经在文章中介绍详细部署的过程：[SpringBoot项目部署到Linux服务器并发布](https://www.cnblogs.com/summerday152/p/13943424.html)。

但是，这种做法有一些问题，就是每次上传jar包到服务器的时候都要很久。为什么呢？也许我们只是创建了一个最基本的SpirngBoot项目，也许我们只是写了三四行代码，最后生成的jar包大小也大的离谱，足足有17M，原因在于生成的可执行jar不仅包含编译之后的类，**还包含了程序运行所依赖的jar包**，而这大部分占比就是这些jar包。

我们通过压缩软件查看生成jar包中的内容：

```yml
slimming.jar # 17M
  - BOOT-INF
	- classes # 存放编译后的class文件 
	- lib # 运行需要的依赖  
	- classpath.idx
 - META-INF # 存放应用相关的辕信息，如MANIFEST.MF
 - org #存放Springboot相关的class文件
```



我们可以明显看到，导致jar包体积那么大的罪魁祸首就是这个lib，且往往我们很少改动这些依赖，频繁改动的往往是一些代码中的逻辑，因此，瘦身的思路就在于：将lib从jar中抽离出来，只需上传一次lib，其他部分的体积就微不足道了，就能够实现秒传。

## 瘦身部署

### 拿到lib目录

按照原先的maven配置方式，在target目录下生成jar。

```xml
    <build>
       <finalName>slimming</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

```shell
$ mvn clean package
```

拿到BOOT-INF/lib目录。

### 改变默认的打包方式

```xml
    <build>
       <finalName>slimming</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <!-- 指定项目的启动类 -->
                    <mainClass>com.hyh.SpringBootSlimmingApplication</mainClass>
                    <!-- 指定打包方式为ZIP -->
                    <layout>ZIP</layout>
                    <includes>
                        <!-- 如果有自己的依赖jar，可以再此导入 -->
                        <include>
                            <groupId>nothing</groupId>
                            <artifactId>nothing</artifactId>
                        </include>
                    </includes>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <!-- 剔除其他的依赖，只保留最简单的结构 -->
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

### 再次打包

```shell
$ mvn clean package
```

此时，`/target`目录下的slimming.jar已经从18000K变成121K。

### 上传lib和jar

```yml
- lib
- slimming.jar
```

```shell
$ nohup java -Dloader.path=./lib -jar slimming.jar $
```

- Dloader.path：告知项目运行依赖的jar包在哪。

## 源码下载

本文内容均为对优秀博客及官方文档总结而得，原文地址均已在文中参考阅读处标注。最后，文中的代码样例已经全部上传至Gitee：[https://gitee.com/tqbx/springboot-samples-learn](https://gitee.com/tqbx/springboot-samples-learn)，另有其他SpringBoot的整合哦。

## 参考阅读

- [【SpringBoot】三十二、SpringBoot项目Jar包如何瘦身部署](https://blog.csdn.net/qq_40065776/article/details/108399327)

