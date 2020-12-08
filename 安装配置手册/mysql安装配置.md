## 下载

下载地址：[https://dev.mysql.com/downloads/mysql/](https://dev.mysql.com/downloads/mysql/)

![image-20201207231610959](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207231610959.png)

选择直接下载

![image-20201207231408006](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207231408006.png)

## 解压

我将Mysql解压到：`E:\devSofts\mysql-8.0.21-winx64\mysql-8.0.21-winx64`地址，后面的内容和这相关，可以根据你的实际情况稍作修改。

## 配置环境变量

此电脑 -> 属性 -> 高级 -> 环境变量。

在Path中添加环境变量：

```plain
E:\devSofts\mysql-8.0.21-winx64\mysql-8.0.21-winx64\bin
```

配置环境变量好处就是，你不必每次都在bin目录下执行命令，你可以在任何位置做相关的操作。

## 新建文件

在`E:\devSofts\mysql-8.0.21-winx64\mysql-8.0.21-winx64`目录下新建`mysql.ini`作为配置文件。

![image-20201207233702955](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207233702955.png)

## 配置mysql.ini

```ini
[mysql]

# 设置mysql客户端默认字符集
default-character-set=utf8 

[mysqld]

#设置3306端口
port = 3306 

# 设置mysql的安装目录
basedir=E:\devSofts\mysql-8.0.21-winx64\mysql-8.0.21-winx64

# 设置mysql数据库的数据的存放目录
datadir=E:\devSofts\mysql-8.0.21-winx64\mysql-8.0.21-winx64\data

# 允许最大连接数
max_connections=200

# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8

# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB

# 允许连接失败的次数。防止有人从该主机试图攻击数据库系统
max_connect_errors=10
```

## 安装

以管理员方式打开cmd

![image-20201207232545659](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207232545659.png)

输入以下命令，正常情况没有反应。

```shell
mysqld --initialize-insecure --user=mysql
```

![image-20201207234126020](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207234126020.png)

接着输入以下命令：

```shell
mysqld install
```

## 启动服务

cmd下输入命令，启动服务。

```shell
net start mysql
```

![image-20201207234149005](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207234149005.png)

## 进入mysql

第一次进入免密登录的：

```shell
mysql -u root -p # 直接回车
```

![image-20201207235114615](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207235114615.png)

修改一下密码，mysql8版本以上的修改密码方式可能有些不同：

```sql
alter user 'root'@'localhost' identified by '123456';
```

接着测试一下，以下命令退出：

```sql
quit;
```

接着登录，使用新密码登录即可。

## 关闭开机自启动

这一步很重要，之后许多需要操作服务的步骤都可以使用：

Win + R，输入：`services.msc`。

![image-20201207235428716](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207235428716.png)

修改为手动开启服务：

![image-20201207235525863](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207235525863.png)

## 连接数据库

之前我是用的Mysql5.5的版本，按照下面这样连接没什么问题。

![image-20201207235711768](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201207235711768.png)

但是连接这次下载mysql8.0版本会出现无法连接的问题：

```plain
出现1251- Client does not support authentication protocol 错误
```

原因在于，Mysql8版本的加密规则是`caching_sha2_password`，不再是原先的`mysql_native_password`。

可以通过将加密规则还原成原先的规则：打开cmd，登录mysql，执行以下命令：

```sql
alter user 'root'@'localhost' identified with mysql_native_password by '123456';
flush privileges; # 刷新权限
```



![image-20201208000705213](img/mysql%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/image-20201208000705213.png)

至此，成功连接。

## 总结

这次安装mysql还是有一点心得体会的，以前总觉得安装配置这玩意对着教程一步一步下去就好了，不需要动脑子。

但是，有时候因为你的某个操作疏忽，又或许是你的版本和教程不同，难免会踩到一些坑。所幸的是，你能很快地从互联网上找到答案，但你想，如果你找不到呢？或者说你的搜索方式不准确呢？

随着学习地不断深入，当我明白每一步操作背后的意图的时候，我会在每一步执行指令按下之前就猜测它可能会发生什么，比如，当你明白环境变量的作用，当你知道如何开启关闭服务，当你知道每一步与下一步之间的联系，你就能自动过滤掉网上一些可能错误的操作，并且更加自信地选择正确的答案。

当我们在做某件事的时候，多想想它为什么这么做，或许下一次你就能比别人更快知道解决问题的方法，你的提问才会更有针对性。

简短的总结，是为了提醒自己，也分享给大家。