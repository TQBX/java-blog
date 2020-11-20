## Docker安装Zookeeper

### 下载并运行

```shell
$ docker search zookeeper # 查看一下镜像

$ docker pull zookeeper:3.4.9  # 拉取指定版本zk镜像

$ docker images  # 查看image ID

$ mkdir -p /root/docker/zookeeper/data
$ docker run -d -p 2181:2181 -v /root/docker/zookeeper/data:/data/ --name zookeeper --privileged 3b83d9104a4c # 最后跟着 image ID

```

### 进入容器

```shell
$ docker ps # 查看zookeeper的CONTAINER ID
$ docker exec -it CONTAINERID /bin/bash  # 后台进入容器
```

### 连接ZooKeeper 服务

```shell
$ cd bin # 进入bin目录
$ ./zkCli.sh
```

![image-20201120170616720](img/docker%E5%AE%89%E8%A3%85zookeeper/image-20201120170616720.png)

### 设置防火墙

关于防火墙，你可以关闭它，或者开启2181端口：

【查看防火墙是否开启】

```bash
$ systemctl status firewalld
```

【开启或关闭防火墙】

```bash
$ systemctl start firewalld
$ systemctl stop firewalld
```

【查看所有开启的端口】

```bash
$ firewall-cmd --list-ports
```

【开启80端口】

```bash
$ firewall-cmd --zone=public --add-port=2181/tcp --permanent
```

【重启防火墙，使其生效】

```bash
$ firewall-cmd --reload
```

### 配置阿里云安全组

来到实例管理页面，点击更多，点击网络和安全组，点击安全组配置。

![image-20201120164723821](img/docker%E5%AE%89%E8%A3%85zookeeper/image-20201120164723821.png)

点击配置规则。

![image-20201120164850301](img/docker%E5%AE%89%E8%A3%85zookeeper/image-20201120164850301.png)

点击添加安全组规则

![image-20201120165048181](img/docker%E5%AE%89%E8%A3%85zookeeper/image-20201120165048181.png)

### 使用Zookeeper图形化客户端工具连接

下载地址：https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip

解压压缩包，进入jar包所在目录，执行命令：

```shell
$ java -jar xxx.jar
```



![image-20201120173806579](img/docker%E5%AE%89%E8%A3%85zookeeper/image-20201120173806579.png)

左上角按钮表示登录，主机地址和端口号：`你的服务器ip:2181`

## Docker常用命令演示

### 查看常用命令help

```dockerfile
[zk: localhost:2181(CONNECTED) 0] help
```

![image-20201120174106981](img/docker%E5%AE%89%E8%A3%85zookeeper/image-20201120174106981.png)

### 创建节点create

通过 `create` 命令在根目录创建了 node1 节点，与它关联的字符串是"node1"

```shell
[zk: localhost:2181(CONNECTED) 0] create /node1 "node1"
```

通过 `create` 命令在根目录创建了 /node1/node1.1 节点，与它关联的内容是数字 123

```shell
[zk: localhost:2181(CONNECTED) 0] create /node1/node1.1 123
```

### 设置节点数据内容set

设置/node1节点的数据内容为"new node!"，此时相当于更新操作。

```shell
[zk: localhost:2181(CONNECTED) 0] set /node1 "new node!"
```

### 获取节点的数据get

`get` 命令可以获取指定节点的数据内容和节点的状态,可以看出我们通过 `set` 命令已经将节点数据内容改为 "new node!"。

```shell
[zk: localhost:2181(CONNECTED) 0] get /node1 #"new node!"
cZxid = 0xb
ctime = Fri Nov 20 09:36:43 GMT 2020
mZxid = 0xd
mtime = Fri Nov 20 09:43:25 GMT 2020
pZxid = 0x10
cversion = 2
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 0
```

下面的一些字段信息，将在本篇第三节znode结构中介绍。

### 查看某个目录的子节点ls

查看根目录下的子节点

```shell
[zk: localhost:2181(CONNECTED) 0] ls /
[node2, zookeeper, node1]
```

查看/node1目录下的子节点

```shell
[zk: localhost:2181(CONNECTED) 0] ls /node1
[node1.1]
```

### 查看节点状态stat

```shell
[zk: localhost:2181(CONNECTED) 0] stat /node1
cZxid = 0xb
ctime = Fri Nov 20 09:36:43 GMT 2020
mZxid = 0xd
mtime = Fri Nov 20 09:43:25 GMT 2020
pZxid = 0xc
cversion = 1
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 1
```

### 查看节点信息和状态ls2

ls2 = ls + stat

```shell
[zk: localhost:2181(CONNECTED) 0] ls2 /node1
[node1.1]
cZxid = 0xb
ctime = Fri Nov 20 09:36:43 GMT 2020
mZxid = 0xd
mtime = Fri Nov 20 09:43:25 GMT 2020
pZxid = 0xc
cversion = 1
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 1
```

### 删除节点delete

删除某一个节点，这个节点必须无子节点。

```shell
[zk: localhost:2181(CONNECTED) 10] delete /node1
Node not empty: /node1
[zk: localhost:2181(CONNECTED) 11] delete /node1/node1.1
[zk: localhost:2181(CONNECTED) 12] get /node1/node1.1
Node does not exist: /node1/node1.1
```

## znode结构

| znode 状态信息 | 解释                                                         |
| -------------- | ------------------------------------------------------------ |
| cZxid          | create ZXID，即该数据节点被创建时的事务 id                   |
| ctime          | create time，即该节点的创建时间                              |
| mZxid          | modified ZXID，即该节点最终一次更新时的事务 id               |
| mtime          | modified time，即该节点最后一次的更新时间                    |
| pZxid          | 该节点的子节点列表最后一次修改时的事务 id，只有子节点列表变更才会更新 pZxid，子节点内容变更不会更新 |
| cversion       | 子节点版本号，当前节点的子节点每次变化时值增加 1             |
| dataVersion    | 数据节点内容版本号，节点创建时为 0，每更新一次节点内容(不管内容有无变化)该版本号的值增加 1 |
| aclVersion     | 节点的 ACL 版本号，表示该节点 ACL 信息变更次数               |
| ephemeralOwner | 创建该临时节点的会话的 sessionId；如果当前节点为持久节点，则 ephemeralOwner=0 |
| dataLength     | 数据节点内容长度                                             |
| numChildren    | 当前节点的子节点个数                                         |

## 参考阅读

- [JavaGuide：Zookeeper实战](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-in-action)