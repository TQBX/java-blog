

## Zookeeper简介

ZooKeeper是Hadoop的正式子项目，它是一个**针对大型分布式系统的可靠协调系统**，提供的功能包括：配置维护、名字服务、分布式同步、组服务等。ZooKeeper的目标就是封装好复杂易出错的关键服务，**将简单易用的接口和性能高效、功能稳定的系统提供给用户**。

Zookeeper是集群的管理者，监视着集群中各个节点的状态根据节点提交的反馈进行下一步合理操作。

## Zookeeper特性

一、顺序一致性【有序性】

从同一个客户端发起的事务请求，最终将会严格地按照其发起顺序被应用到Zookeeper中去。

有序性是Zookeeper中非常重要的一个特性：

- 所有的更新都是全局有序的，每个更新都有一个唯一的时间戳，这个时间戳称为zxid（Zookeeper Transaction Id）。
- 但，读请求只会相对于更新有序，也就是读请求的返回结果中会带有这个Zookeeper最新的zxid。

二、原子性

所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，即整个集群要么都成功应用了某个事务，要么都没有应用。

三、单一视图

无论客户端连接的是哪个 Zookeeper 服务器，其**看到的服务端数据模型都是一致的**。

四、可靠性

一旦服务端成功地应用了一个事务，并完成对客户端的响应，那么该事务所引起的服务端状态变更将会一直被保留，除非有另一个事务对其进行了变更。

五、实时性

Zookeeper 保证在一定的时间段内，客户端最终一定能够从服务端上读取到最新的数据状态。

## Windows环境下安装Zookeeper

下载地址：[https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz](https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.6.2/apache-zookeeper-3.6.2-bin.tar.gz)

> 未完待续。。。



## 参考阅读

- [芋道源码：芋道Zookeeper极简入门](http://www.iocoder.cn/Zookeeper/install/ )

- [Windows下ZooKeeper的配置和启动步骤——单机模式](https://www.jianshu.com/p/66857cbccbd3)