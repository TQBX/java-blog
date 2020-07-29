# maven添加本地包命令mvn install:install-file各个属性详解

转载于：[http://www.yayihouse.com/yayishuwu/chapter/1415](http://www.yayihouse.com/yayishuwu/chapter/1415)

## 添加带有别名的jar包

如：common-auth-0.0.1-SNAPSHOT-core.jar，别名为core。maven指令如下：

```shell
mvn install:install-file 
-Dfile=F:/common-auth/target/common-auth-0.0.1-SNAPSHOT-core.jar 
-DgroupId=com.cloud 
-DartifactId=common-auth 
-Dversion=0.0.1-SNAPSHOT 
-Dclassifier=core  
-Dpackaging=jar
```

## 添加没带别名的jar包

如：common-auth-0.0.1-SNAPSHOT.jar，maven指令如下：

```shell
mvn install:install-file 
-Dfile=F:/common-auth/target/common-auth-0.0.1-SNAPSHOT.jar 
-DgroupId=com.cloud 
-DartifactId=common-auth 
-Dversion=0.0.1-SNAPSHOT 
-Dpackaging=jar
```

## 属性详解

`-Dfile`：包的本地真实地址

`-DgroupId`：pom.xml中groupId

`-DartifactId`：pom.xml中artifactId

`-Dversion`：pom.xml中0.0.1-SNAPSHOT

`-Dpackaging`：jar或war，包的后缀名

`-Dclassifier`：兄弟包的别名，也就是-Dversion值后面的字符common-auth-0.0.1-SNAPSHOT-core.jar的-core