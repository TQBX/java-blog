# JDK命令行工具

这些命令在JDK安装目录的bin目录下：

- **`jps`** (JVM Process Status）: 类似 UNIX 的 `ps` 命令。用户查看所有 Java 进程的启动类、传入参数和 Java 虚拟机参数等信息；
- **`jstat`**（ JVM Statistics Monitoring Tool）: 用于收集 HotSpot 虚拟机各方面的运行数据;
- **`jinfo`** (Configuration Info for Java) : Configuration Info forJava,显示虚拟机配置信息;
- **`jmap`** (Memory Map for Java) :生成堆转储快照;
- **`jhat`** (JVM Heap Dump Browser ) : 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果;
- **`jstack`** (Stack Trace for Java):生成虚拟机当前时刻的线程快照，线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合。

## jps ： 查看所有Java进程

`jps`：显示虚拟机执行主类名称以及这些进程的本地虚拟机唯一 ID（Local Virtual Machine Identifier,LVMID）

```java
D:\Java_Project\springboot>jps
13940 RemoteMavenServer
10216
14156 Jps
```

- `jps -q` ：只输出进程的本地虚拟机唯一 ID。

```java
D:\Java_Project\springboot>jps -q
13940
9636
10216
```

- `jps -l`：输出主类的全名，如果进程执行的是 Jar 包，输出 Jar 路径。

```java
D:\Java_Project\springboot>jps -l
13940 org.jetbrains.idea.maven.server.RemoteMavenServer
10216
17272 sun.tools.jps.Jps
```

- `jps -v`：输出虚拟机进程启动时 JVM 参数。
- `jps -m`：输出传递给 Java 进程 main() 函数的参数。

## jstat：监控虚拟机各种运行状态信息

`jstat（JVM Statistics Monitoring Tool）` 使用于监视虚拟机各种运行状态信息的命令行工具。 它可以显示本地或者远程（需要远程主机提供 RMI 支持）虚拟机进程中的类信息、内存、垃圾收集、JIT 编译等运行数据，在没有 GUI，只提供了纯文本控制台环境的服务器上，它将是运行期间定位虚拟机性能问题的首选工具。

**格式**：`jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]`

比如 `jstat -gc -h3 31736 1000 10`表示分析进程 id 为 31736 的 gc 情况，每隔 1000ms 打印一次记录，打印 10 次停止，每 3 行后打印指标头部。

常见的**options**：[https://www.cnblogs.com/sxdcgaq8080/p/11089841.html](https://www.cnblogs.com/sxdcgaq8080/p/11089841.html)

- `jstat -class vmid` ：显示 ClassLoader 的相关信息；

  - ```java
    jstat -class 13940
    Loaded  Bytes  Unloaded  Bytes     Time
      4028  7463.7        0     0.0       7.04	
    ```

  - loaded：加载class的数量

  - Bytes：所占用空间大小【kb】

  - Unloaded：未加载数量

  - Bytes：未加载占用空间

  - Time：时间

- `jstat -compiler vmid` ：显示 JIT 编译的相关信息；
- `jstat -gc vmid` ：显示与 GC 相关的堆信息；
- `jstat -gccapacity vmid` ：显示各个代的容量及使用情况；
- `jstat -gcnew vmid` ：显示新生代信息；
- `jstat -gcnewcapcacity vmid` ：显示新生代大小与使用情况；
- `jstat -gcold vmid` ：显示老年代和永久代的行为统计，从jdk1.8开始,该选项仅表示老年代，因为永久代被移除了；
- `jstat -gcoldcapacity vmid` ：显示老年代的大小；
- `jstat -gcpermcapacity vmid` ：显示永久代大小，从jdk1.8开始,该选项不存在了，因为永久代被移除了；
- `jstat -gcutil vmid` ：显示垃圾收集信息；
- `jstat -gcmetacapacity vmid`：jdk1.8之后元数据空间的统计
- `jstat -printcompilation vmid`：jvm编译方法统计

## jinfo：实时查看和调整虚拟机各项参数

- `jinfo vmid` :输出当前 jvm 进程的全部参数和系统属性 (第一部分是系统的属性，第二部分是 JVM 的参数)。
- `jinfo -flag name vmid` :输出对应名称的参数的具体值。
  - 如` MaxHeapSize`：查看当前 jvm 进程
  - 是否开启打印 GC 日志 ( `-XX:PrintGCDetails` :详细 GC 日志模式
  - `MaxTenuringThreshold`，新生代晋升老年代的年龄阈值。

```java

$ jinfo -flag MaxHeapSize 10216
-XX:MaxHeapSize=1180696576


$ jinfo -flag PrintGC 10216
-XX:-PrintGC


$ jinfo -flag MaxTenuringThreshold 10216
-XX:MaxTenuringThreshold=6

```

- `jinfo -flag [+|-]name vmid` 开启或者关闭对应名称的参数。

```java
C:\Users\SnailClimb>jinfo  -flag  PrintGC 17340
-XX:-PrintGC

C:\Users\SnailClimb>jinfo  -flag  +PrintGC 17340

C:\Users\SnailClimb>jinfo  -flag  PrintGC 17340
-XX:+PrintGC
```

## jmap：生成堆转储快照

`jmap`（Memory Map for Java）命令用于生成堆转储快照。 如果不使用 `jmap` 命令，要想获取 Java 堆转储，可以使用 `“-XX:+HeapDumpOnOutOfMemoryError”` 参数，可以让虚拟机在 OOM 异常出现之后自动生成 dump 文件，Linux 命令下可以通过 `kill -3` 发送进程退出信号也能拿到 dump 文件。

`jmap` 的作用并不仅仅是为了获取 dump 文件，它还可以查询 finalizer 执行队列、Java 堆和永久代的详细信息，如空间使用率、当前使用的是哪种收集器等。和`jinfo`一样，`jmap`有不少功能在 Windows 平台下也是受限制的。



## jhat：分析heapdump文件

**`jhat`** 用于分析 heapdump 文件，它会建立一个 HTTP/HTML 服务器，让用户可以在浏览器上查看分析结果。

## jstack：生成虚拟机当前时刻的线程快照

`jstack`（Stack Trace for Java）命令用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合.

生成线程快照的目的主要是定位线程长时间出现停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的原因。线程出现停顿的时候通过`jstack`来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者在等待些什么资源。

# JDK可视化分析工具

JConsole：是基于 JMX 的可视化监视、管理工具。可以很方便的监视本地及远程服务器的 java 进程的内存使用情况。你可以在控制台输出`console`命令启动或者在 JDK 目录下的 bin 目录找到`jconsole.exe`然后双击启动。

# Visual VM：多合一故障处理工具



# Arthas

```
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

```
[arthas@1565]$ dashboard
```

一个Java进程CPU突然飙升，如何定位问题？

1. top -Hp  jstack

2. arthas dashboard   直接观测到cpu占比，  thread threa ID

- 业务线程：查看业务
- GC线程：读GC日志

死锁怎么排查？

1. jstack 观察线程情况
2. arthas thread -b