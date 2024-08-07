[toc]

本文摘自：《JavaGuide》

# Redis简介

Redis数据库，与传统数据库不同，Redis的数据存储在内存中，读写速度非常快。

Redis可以应用的方向：缓存，分布式锁，事务，持久化，LUA脚本，LRU驱动事件，多种集群方案。

# 为什么要用Redis

**高性能：（基于内存、访问速度快）**

- 假设数据存储在数据库中，从中读取其实是从硬盘上读取的，读取速度相对较慢。
- 但将其存储在缓存中，再读取的时候，直接可以从缓存中获取，直接操作内存，速度很快。
- 如果数据库中的对应数据改变之后，需要同步改变缓存中响应的数据。

**高并发：（mysql的qps在4k左右，4c8g，使用Redis达到5w+到10w+，单机情况）**

- 直接操作缓存能够承受的请求远远大于直接访问数据库，所以可以把数据库中的部分数据转移到缓存中去，这样用户的一部分请求将会直接到缓存，而不经过数据库。

**功能全面：**

> 在项目中的使用场景：
>
> 1. **setnx 实现分布式锁**，优化独占锁为分段锁，过期时间设置为活动结束时间
> 2. 用于缓存：string、hash、list、zset
> 3. 实时消息控制（**消息队列**）：阻塞队列 blpush blpop
> 4. 延迟队列：Redisson内置延迟对立额，基于zset实现
> 5. 分布式session
> 6. 限流…… （Redis+lua 或 RRateLimiter：lua + 令牌桶算法）



# 为什么不用map做缓存

缓存分为本地缓存和分布式缓存。

Map：

- 是本地缓存，轻量及快速，生命周期随jvm的销毁而结束。
- 在多实例的情况下，每个实例都需要保存一份缓存，缓存不具有一致性。

Redis或memcached：

- 是分布式缓存，在多实例的情况下，各实例共用一份缓存，缓存具有一致性。
- 需要保持分布式缓存服务的高可用，整个程序架构上较为复杂。

# Redis为什么这么快？

1. 基于内存，内存访问速度>>磁盘，并且利用自定义内存分配器jemalloc有效处理内存 
2. 非阻塞IO多路复用技术为中心的事件驱动程序设计模型，读写请求不阻塞其他操作。
3. 数据结构都对内存做了优化：比如：小值编码等
   1. SDS紧凑型字符串表示 string（动态string）
   2. 压缩列表zip list：list，sorted set
   3. 跳表skiplist：zset
4. 持久化操作也是异步的，不回阻塞主线程读写
5. 管道技术，一次性发送多个命令，减少网络往返
6. 客户端缓存、减少对Redis服务器的访问频率

![why-Redis-so-fast](img/Redis%E5%AD%A6%E4%B9%A0/why-Redis-so-fast-TbWX24ja.png)

# Redis线程模型

[https://www.javazhiyin.com/22943.html](https://www.javazhiyin.com/22943.html)

Redis内部使用文件事件处理器`filter event handler`，这个**文件处理器是单线程**的，因此Redis被称为单线程的模型。

为什么选择**单线程网络模型**？因为CPU通常不会成为性能瓶颈，瓶颈往往是内存和网络，因此单线程就够了。

![img](img/redis%E5%AD%A6%E4%B9%A0/single-threaded-redis.png)

Redis1.0到6.0之前，核心网络模型是：单Reactor模型：**采用epoll/select/kqueue等IO多路复用技术同时监听多个socket，在单线程的事件循环中不断去处理事件（客户端请求），最后回写响应数据到客户端**。本质就是：I/O多路复用 + 非阻塞I/O

多个socket可能会并发产生不同的操作，每个操作对应不同的文件事件，但是IO多路复用程序会监听多个socket，会将socket产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。

对于那些想利用多核优势提升性能的用户来说，redis官方给出的解决方案是：在同一个机器上多跑几个redis实例，事实上，为了保证高可用，往往利用redis分布式集群多节点和数据分片负载均衡来提升性能和保证高可用。

！！！实际上，参考文章：[Redis真的是单线程？](https://strikefreedom.top/archives/multiple-threaded-network-model-in-redis)

上面说的单线程的是网络模型，实际上，redis4.0开始就引入了多线程处理异步任务，来处理一些比较耗时的命令，将命令异步化，避免阻塞单线程的事件循环。（例子，超大key删除，会导致单线程阻塞好几秒，阻塞其他事件！）

**于是，redis 4.0之后加入一些非阻塞的命令：unlink，flushall async， flushdb async**

# 为什么引入多线程？

之前选择单线程是因为CPU不是性能瓶颈，内存和网络才是，但是现在I/O瓶颈越来越明显。Redis 的单线程模式会导致系统消耗很多 CPU 时间在网络 I/O 上从而降低吞吐量，要提升 Redis 的性能有两个方向：
- 优化网络I/O模块
- 提高机器内存读写的速度（依赖硬件，难）

I/O怎么优化：
- 零拷贝（什么是零拷贝） or DPDK（旁路网卡I/O绕过内核协议栈）（不适配or太复杂）
- 利用多核！！！性价比最高的方案！因此！引入多线程IO

Redis6.0之后在网络模型中实现I/O多线程，提高网络IO读写性能。从Reactor进化为Multi-Reactors模式：

![img](img/redis%E5%AD%A6%E4%B9%A0/multi-reactors.png)

从单线程事件循环 变成：多个线程（sub reactors）各自维护一个独立的事件循环，由main reactor负责接收新连接并分发给sub reactors去独立处理，最后sub reactors回写响应给客户端。

> 缺陷：在 Redis 的多线程方案中，I/O 线程任务仅仅是通过 socket 读取客户端请求命令并解析，却没有真正去执行命令，所有客户端命令最后还需要回到主线程去执行，因此对多核的利用率并不算高，而且每次主线程都必须在分配完任务之后忙轮询等待所有 I/O 线程完成任务之后才能继续执行其他逻辑。
>
> Redis 目前的多线程方案更像是一个折中的选择：既保持了原系统的兼容性，又能利用多核提升 I/O 性能。

# Redis和memcached的区别/为什么不用m

| 对比参数     | Redis                                                        | Memecached                                                 |
| ------------ | ------------------------------------------------------------ | ---------------------------------------------------------- |
| 类型         | 1、支持内存<br />2、非关系型数据库                           | 1、支持内存<br />2、key-value键值对的形式<br />3、缓存系统 |
| 数据存储类型 | String、List、Set<br />Hash、SortSet（Zset）                 | 1、文本型<br />2、二进制类型                               |
| 操作类型     | 1、批量操作<br />2、事务支持（假事务）<br />3、每个类型不同的CRUD | 1、CRUD<br />2、少量的其他命令                             |
| 附加功能     | 1、发布订阅模式<br />2、主从分区<br />3、序列化支持<br />4、脚本支持（LUA脚本） | 多线程服务支持                                             |
| 网络IO模型   | 单线程模型                                                   | 多线程、非阻塞IO模式                                       |
| 事件库       | 自动简易事件库AeEvent                                        | 贵族血统的LibEvent事件库                                   |
| 持久化支持   | RDB、AOF                                                     | 不支持                                                     |

# Redis常见数据结构以及使用场景分析

## String

常用操作：set、get、decr、incr、mget

String数据结构是简单的key-value类型，应用于常规的缓存应用：微博数、粉丝数等。

> 【项目】：库存扣减！decr

## Hash

常用操作：hget、hset、hgetall

String类型的field和value的映射表，hash适合用于存储对象，后续操作时，可以直接仅仅修改该对象中的某个字段的值。如：

```java
key=user122
value={
    "id": 1,
    "name":"summerday",
    "age":20
}
```

> 【项目】：存储抽奖算法需要的缓存

## List

常用操作：lpush、rpush、lpop、rpop、lrange

双向链表，可以支持双向查找遍历，可以实现消息列表，粉丝关注列表的，lrange命令可以实现从某个元素开始读取多少个元素，可以实现简单的高性能分页。

> 【项目】：阻塞队列的实现，blpop，blpush，

## Set

常用操作：sadd，spop，smembers，sunion

提供类似list的功能，但是set中元素自动去重，另外交集并集差集的操作可以轻松实现共同关注、共同粉丝的功能。

## Sorted Set

常用操作：zadd、zrange、zrem、zcard

在set的基础上增加了一个权重系数score，使得集合中的元素能够按score进行有序排列，可以实现各种排行榜。

> 延迟队列，更新库存，减少数据库的压力，score是时间戳
>
> 原理：基于选择机制，根据存储元素的长度or大小，选择压缩列表（类似于数组，双向链表）或者跳表（链表基础上+多级索引，logn时间查找、删除、插入，多个层级的链表组成）

跳表、平衡树、红黑树、B+树的区别？

- 平衡树：平衡树的插入、删除和查询的时间复杂度和跳表一样都是 **O(log n)**，范围查询也可以达到一样的效果，但是，但是它的每一次插入或者删除操作都需要保证整颗树左右节点的绝对平衡，只要不平衡就要通过旋转操作来保持平衡，这个过程是比较耗时的。（跳表使用概率平衡）
- 红黑树：跳表的实现也更简单一些，不需要通过旋转和染色（红黑变换）来保证黑平衡。
- B+树：跳表不需要存储大量数据，跳表实现 zset 时相较前者来说更简单一些，在进行插入时只需通过索引将数据插入到链表中合适的位置再随机维护一定高度的索引即可，也不需要像 B+树那样插入时发现失衡时还需要对节点分裂与合并。

## HyperLogLog、Bitmap、Geospatial，Bloom filter

HyperLogLog（基数统计）、Bitmap （位图）、Geospatial (地理位置)

Bitmap：存储连续的二进制0or1，bitmap极大节省存储空间，统计活跃用户：SETBIT 20220329 [id] 1 设置id用户2022年3月29日活跃为1。

BITOP and desk1 20220329
(integer) 1
BITCOUNT desk1
(integer) 1

Redis v4.0 之后有了 Module（模块/插件） 功能，Redis Modules 让 Redis 可以使用外部模块扩展其功能 。布隆过滤器就是其中的 Module。详情可以查看 Redis 官方对 Redis Modules 的介绍：https://redis.io/modules

# Redis实现分布式锁



# Redis做消息队列

Redis2.0之前，使用List中的rpush和lpop实现简易版消息队列（问题是会不断轮询调用指令，存在性能问题）

以及阻塞式的BLPOP和BRPOP这种阻塞式读取的命令，

```redis
# 超时时间为 10s
# 如果有数据立刻返回，否则最多等待10秒
BRPOP myList 10
null
```

Redis2.0引入了发布订阅（pub/sub），解决了list实现消息队列没有广播机制的问题：publisher，channel，subscriber

- 发布者通过publish 发送消息到指定channel
- 订阅者通过subscribe订阅channel，可以订阅一个or多个channel

> 广播、单播都支持，还支持简单正则匹配，但是消息丢失，消息堆积等问题不能很好解决。

Redis5.0引入Stream来做消息队列：

![img](img/redis%E5%AD%A6%E4%B9%A0/redis-stream-structure.png)

- 发布 / 订阅模式
- 按照消费者组进行消费（借鉴了 Kafka 消费者组的概念）
- 消息持久化（ RDB 和 AOF）
- ACK 机制（通过确认机制来告知已经成功处理了消息）
- 阻塞式获取消息

这是一个有序的消息链表，每个消息都有一个唯一的 ID 和对应的内容。ID 是一个时间戳和序列号的组合，用来保证消息的唯一性和递增性。内容是一个或多个键值对（类似 Hash 基本数据类型），用来存储消息的数据。

这里再对图中涉及到的一些概念，进行简单解释：

- `Consumer Group`：消费者组用于组织和管理多个消费者。消费者组本身不处理消息，而是再将消息分发给消费者，由消费者进行真正的消费
- `last_delivered_id`：标识消费者组当前消费位置的游标，消费者组中任意一个消费者读取了消息都会使 last_delivered_id 往前移动。
- `pending_ids`：记录已经被客户端消费但没有 ack 的消息的 ID。

总结：Redis可以做简易的消息队列，但是不建议使用，因为有更成熟的方案：RocketMQ、Kafka等。

# Redis实现延迟任务

场景：订单在10分钟后未支付就失效，红包24小时未被查收就自动退还。

方案：

1. redis过期事件监听：时效性差、丢消息、多服务实例下消息重复消费。
2. Redisson内置的延迟队列：减少丢消息的可能（即使redis宕机了，也可以扫描数据库的方法作为补偿），消息不存在重复消费。

Redisson 的延迟队列 RDelayedQueue 是基于 Redis 的 SortedSet 来实现的。SortedSet 是一个有序集合，其中的每个元素都可以设置一个分数，代表该元素的权重。Redisson 利用这一特性，将需要延迟执行的任务插入到 SortedSet 中，并给它们设置相应的过期时间作为分数。

Redisson 使用 `zrangebyscore` 命令扫描 SortedSet 中过期的元素，然后将这些过期元素从 SortedSet 中移除，并将它们加入到就绪消息列表中。就绪消息列表是一个阻塞队列，有消息进入就会被监听到。这样做可以避免对整个 SortedSet 进行轮询，提高了执行效率。

# Redis过期策略

redis用于缓存，既然是缓存，就是用的内存，内存是宝贵的，不可能用来当存储工具用。那么写入数据太多了咋办？这就涉及到过期策略，redis是怎么处理过期数据的？

而Redis拥有设置过期时间的功能，在set key的时候，设置一个expire time，通过这个过期时间指定key存活时间。

**Redis的过期策略：定期删除 + 惰性删除**

- 定期删除：<u>每隔100ms随机抽取一些设置了过期时间的key，检查他们是否过期，如果过期就删除</u>。注：随机抽取是为了防止key数量过多而导致的cpu负载。

- 惰性删除：定期删除可能会导致许多过期的key到了时间并没有被删除掉，这时就有了惰性删除，如果我们设置的过期key到了指定的时间还没有被删除掉，他还停留在内存中，<u>除非系统去查那个key</u>，那个key才会被Redis删除。

假如我们设置了大量过期时间的key，定期删除时漏掉了许多key，系统也没有及时去查这个key，那么将会导致一种严重的情况：**大量过期的key堆积在内存中，导致Redis内存耗尽**。

这时就需要用到Redis的**内存淘汰**机制。

# Redis提供数据淘汰策略

- **volatile-lru：从已设置过期时间的数据集中挑选最近最少使用的数据，淘汰之。**

> 手写一个lru算法！！！
>
> 单链表思路：
>
> 1. 维护一个链表，头部表示最晚访问的，尾部表示最先访问的（先走）
> 2. 移除操作：移除尾部即可
> 3. 添加操作：加入的key存在，更新一下value，扔到头部（移除当前的，在头部加入新的），如果key不存在，就新创建一个，加入到头部，加入之后，如果cap超过了阈值，就移除尾部
>
> 这个思路没问题，但是可以优化：
>
> 1. 可以用双向链表快速找到某个节点的前向节点，快速删除节点
> 2. 可以用map存储每个key 和对应的node
>
> ```java
> import java.util.HashMap;  
> import java.util.Map;  
>   
> class ListNode {  
>     int key;  
>     int value;  
>     ListNode prev;  
>     ListNode next;  
>   
>     ListNode(int k, int v) {  
>         key = k;  
>         value = v;  
>     }  
> }  
>   
> class LRUCache {  
>     private int capacity;  
>     private Map<Integer, ListNode> map;  
>     private ListNode head;  
>     private ListNode tail;  
>   
>     public LRUCache(int capacity) {  
>         this.capacity = capacity;  
>         map = new HashMap<>();  
>         head = new ListNode(0, 0);  //虚拟头节点
>         tail = new ListNode(0, 0);  //虚拟尾节点
>         head.next = tail;  
>         tail.prev = head;  
>     }  
>   
>     public int get(int key) {  
>         ListNode node = map.get(key);  
>         if (node == null) {  
>             return -1; // key not found  
>         }  
>         // move node to head  
>         moveToHead(node);  
>         return node.value;  
>     }  
>   
>     public void put(int key, int value) {  
>         ListNode node = map.get(key);  
>         if (node == null) {  
>             // new key-value pair  
>             ListNode newNode = new ListNode(key, value);  
>             // add to map  
>             map.put(key, newNode);  
>             // add to linked list  
>             addToHead(newNode);  
>             if (map.size() > capacity) {  
>                 // remove tail  
>                 removeNode(tail.prev);  
>             }  
>         } else {  
>             // update value  
>             node.value = value;  
>             // move node to head  
>             moveToHead(node);  
>         }  
>     }  
>   
>     private void addToHead(ListNode node) {  
>         node.prev = head;  
>         node.next = head.next;  
>         head.next.prev = node;  
>         head.next = node;  
>     }  
>   
>     private void removeNode(ListNode node) {  
>         map.remove(node.key);  
>         node.prev.next = node.next;  
>         node.next.prev = node.prev;  
>     }  
>   
>     private void moveToHead(ListNode node) {  
>         removeNode(node);  
>         addToHead(node);  
>     }  
> }  
>   
> // Example usage  
> public class Main {  
>     public static void main(String[] args) {  
>         LRUCache cache = new LRUCache(2);  
>   
>         cache.put(1, 1);  
>         cache.put(2, 2);  
>         System.out.println(cache.get(1));       // returns 1  
>         cache.put(3, 3);                        // evicts key 2  
>         System.out.println(cache.get(2));       // returns -1 (not found)  
>         cache.put(4, 4);                        // evicts key 1  
>         System.out.println(cache.get(1));       // returns -1 (not found)  
>         System.out.println(cache.get(3));       // returns 3  
>         System.out.println(cache.get(4));       // returns 4  
>     }  
> }
> 
> ```
>
> 当然了，其实可以继承LinkedHashMap简易操作
>
> ```java
> public class LRUCache<K, V> extends LinkedHashMap<K, V> {
>     private int capacity;
> 
>     /**
>      * 传递进来最多能缓存多少数据
>      *
>      * @param capacity 缓存大小
>      */
>     public LRUCache(int capacity) {
>         super(capacity, 0.75f, true);
>         this.capacity = capacity;
>     }
> 
>     /**
>      * 如果map中的数据量大于设定的最大容量，返回true，再新加入对象时删除最老的数据
>      *
>      * @param eldest 最老的数据项
>      * @return true则移除最老的数据
>      */
>     @Override
>     protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
>         // 当 map中的数据量大于指定的缓存个数的时候，自动移除最老的数据
>         return size() > capacity;
>     }
> }
> 
> ```



- volatile-ttl：从已设置过期时间的数据集中挑选将要过期的数据淘汰。
- volatile-random：从已设置过期时间的数据集中任意选择数据淘汰。
- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key。（最常用）
- allkeys-random：从数据集中任意选择数据淘汰。
- no-eviction：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。

4.0版本之后新增：

- volatile-lfu：从已设置过期时间的数据集中挑选最不经常使用的数据淘汰。
- allekeys-lfu：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的key。

# Redis的持久化机制RDB和AOF

Redis支持两种持久化的操作，而memcached并不支持持久化操作。

- RDB，快照：Redis可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本，Redis创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（主从结构），还可以将快照留在原地以便重启服务器时使用。

> 快照持久化时Redis默认采用的持久化方式。

- AOF，只追加文件，append-only file：实时性更好，已成为主流的持久化方案。默认情况下Redis没有开启，需要通过appendonly yes开启。

  开启AOF持久化之后，每执行一条会更改Redis中的数据的命令，Redis就会将该命令写入硬盘中的AOF文件。

  Redis在配置文件中存在三种不同的AOF持久化方式：

  ```conf
  appendfsync always # 每次有数据修改发生时，都会写入AOF文件，这样会严重降低Redis的速度
  appendfsync everysec # 每秒钟同步一次，显式地将多个写命令同步到磁盘
  appendfsync no # 让操作系统决定合适进行同步
  ```

  为了兼顾数据和写入性能，可以采用第二个。
  
- RDB 和 AOF 的混合持久化(Redis 4.0 新增)

总结：

- Redis 保存的数据丢失一些也没什么影响的话，可以选择使用 RDB。
- 不建议单独使用 AOF，因为时不时地创建一个 RDB 快照可以进行数据库备份、更快的重启以及解决 AOF 引擎错误。
- 如果保存的数据要求安全性比较高的话，建议同时开启 RDB 和 AOF 持久化或者开启 RDB 和 AOF 混合持久化。

# Redis事务

Redis通过MULTI、EXEC、WATCH等命令来实现事务功能。

在传统的关系型数据库中，常常用ACID来检验事务功能的可靠性和安全性。

在Redis中，事务总是具有原子性（Atomicity）、一致性（Consistency）和隔离性（Isolation），并且在Redis运行在某种特定的持久化模式下，事务也具有持久性（Durability）。

# Redis缓存雪崩

缓存在同一时间大面积的失败，后面的请求都会落到数据库，造成数据库短时间内承受大量请求而崩掉。

【解决方案】

- 事前：尽量保证整个Redis集群的高可用性，发现机器宕机尽快补上。选择合适的内存淘汰策略。
- 事中：本地echache缓存+hystrix限流&降级，避免Mysql崩掉。
- 事后：利用Redis持久化机制保存的数据尽快恢复缓存。

# 缓存穿透

大量请求的key根本不存在于缓存中，导致请求直接到了数据库上，根本没有经过缓存这一层。

可以通过`show variables like '%max_connections%';`命令查看MySQL的最大连接数，如果并发请求过多，数据库很容易挂掉。

【解决方案】

- 做好参数校验，一些不合法的参数请i去直接抛出异常信息返回给客户端。

- 缓存无效key：如果缓存和数据库都查不到某个key的数据，就写一个到Redis中并设置过期时间，这种方式可以解决请求的key变化不频繁的情况，如果请求的key变化频繁，这种方案便不能解决这个问题。注，通常key的形式为：表名:列名:主键名:主键值

  ```java
  public Object getObjectInclNullById(Integer id) {
       // 从缓存中获取数据
       Object cacheValue = cache.get(id);
       // 缓存为空
       if (cacheValue == null) {
           // 从数据库中获取
           Object storageValue = storage.get(key);
           // 缓存空对象
           cache.set(key, storageValue);
           // 如果存储数据为空，需要设置⼀个过期时间(300秒)
           if (storageValue == null) {
               // 必须设置过期时间，否则有被攻击的⻛险
               cache.expire(key, 60 * 5);
           }
           return storageValue;
       }
       return cacheValue; 
  }
  ```

  

- 使用**布隆过滤器**：通过它判断一个给定的数据是否存在于海量数据库中，当用户请求过来，判断用户发来请求的值是否存在于布隆过滤器中，不存在的话，直接返回请求参数信息给客户端。

# 如何保证缓存与数据库双写时数据一致性

1. 读请求和写请求串行化，串到一个内存队列中，可以保证一致性，但会导致系统的吞吐量大幅度降低。
2. Cache Aside Pattern
   - 读的时候，先读缓存，缓存没有的话，就读数据库，然后取出数据后放入缓存，同时返回响应。
   - 更新的时候，先更新数据库，然后再删除缓存。这样会存在一个问题，如果更新数据库之后，再删除缓存，缓存删除失败，这时缓存中是旧数据，而数据库中是新数据，这时就出现了数据不一致。
     - 解决思路：先删除缓存，再更新数据库，如果数据库更新失败了，那么数据库中是旧数据，缓存中为空，读缓存的时候，缓存为空，自然就去数据库中读了旧数据，然后更新到缓存中，数据一致。
     - 这时候又出现了问题，如果先删除了缓存，还没来得及更新数据库，一个请求过来，去都缓存，发现缓存空了，去查数据库，查到了修改前的数据，放到缓存中，随后数据变更的程序完成了数据库的修改，这时又出现了数据不一致。（高并发环境下出现该问题）
     - 解决思路：更新数据的时候，根据数据的唯一标识，将操作路由之后，发送到一个jvm内部队列中，读取数据的时候，如果发现数据不再缓存中，那么重新执行：读取数据+更新缓存的操作，根据唯一标识路由之后，也发送到同一个jvm内部队列中。一个队列对应一个工作线程，每个工作线程串行拿到对应的操作，然后一条条地执行，如此，一个数据变更的操作，先删除缓存，然后再去更新数据库，但是还没完成更新，如果此时一个读请求过来，没有读到缓存，可以先将缓存更新的请求发送到队列中，此时会在队列中积压，然后同步等待缓存更新完成。
     - [https://github.com/doocs/advanced-java/blob/master/docs/high-concurrency/Redis-consistence.md](https://github.com/doocs/advanced-java/blob/master/docs/high-concurrency/Redis-consistence.md)

