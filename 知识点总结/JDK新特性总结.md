# Java 8

interface方法可以用default和static修饰

functional interface 函数式接口，有且只有一个抽象方法，可以有多个非抽象方法的接口

[Lambda表达式](Java基础/Java8/Lambda表达式.md)、[StreamAPI](Java基础/Java8/StreamAPI.md)、[Optional](Java基础/Java8/Optional.md)、[日期API](Java基础/Java8/日期API.md)

# Java 9

Jshell、模块化系统

G1成为默认的垃圾回收器（java8默认是并行的新生代parallel scavenge和parallel old老年代） java9，G1（grabage-first garbage collector）成为默认的垃圾回收器

快速创建不可变集合：List of(), Set of(), Map of()

String存储结构优化：char[] 改为 byte[]

接口私有方法：（接口内复用）

try with resources增强

Stream & Optional增强

增加了进程接口：java.lang.ProcessHandle

响应式流（Reactive Streams）

变量句柄

# Java 11

ZGC. ZGC 也采用标记-复制算法

GC 停顿时间不超过 10ms

即能处理几百 MB 的小堆，也能处理几个 TB 的大堆

应用吞吐能力不会下降超过 15%（与 G1 回收算法相比）

方便在此基础上引入新的 GC 特性和利用 colored 针以及 Load barriers 优化奠定基础

当前只支持 Linux/x64 位平台

# Java 14

ZGC 转正 java -XX:+UseZGC className

# Java19

虚拟线程：轻量级线程

lua、python里叫 Coroutine， go里面叫Goroutine，协程！

java中之前：平台线程，是对操作系统线程的包装，需要在用户空间和内核空间进行上下文切换，线程数量由操作系统线程数量决定。

thread per request，servlet容器为每个请求都创建一个thread处理请求，比如读取数据库，需要网络IO，cpu空闲，但是线程并没有被释放。高并发情况下，cpu使用率低，但是占用线程多！

线程资源被耗尽，无法处理请求了，但是cpu使用率还是很低。

线程池只是减少创建和销毁的开销，并不会提升上限，上限由操作系统决定。

为了提高并发量，采用异步编程（非阻塞模型）处理，学习成本高，不便于调试维护。

虚拟线程应运而生。

以平台线程为载体的轻量级线程，占用资源更少。

Thread.ofVirtual().start()

<img src="img/JDK%E6%96%B0%E7%89%B9%E6%80%A7%E6%80%BB%E7%BB%93/image-20240809185832649.png" alt="image-20240809185832649" style="zoom:50%;" />

虚拟线程（Virtual Thread-）是 JDK 而不是 OS 实现的轻量级线程(Lightweight Process，LWP），许多虚拟线程共享同一个操作系统线程，虚拟线程的数量可以远大于操作系统线程的数量。

![image-20240809185909415](img/JDK%E6%96%B0%E7%89%B9%E6%80%A7%E6%80%BB%E7%BB%93/image-20240809185909415.png)

虚拟线程在其他多线程语言中已经被证实是十分有用的，比如 Go 中的 Goroutine、Erlang 中的进程。

虚拟线程避免了上下文切换的额外耗费，兼顾了多线程的优点，简化了高并发程序的复杂，可以有效减少编写、维护和观察高吞吐量并发应用程序的工作量。



虚拟线程比起goroutine，[协程](https://www.zhihu.com/search?q=协程&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2664084546})什么，要好理解得多，看这名字就大概知道它在做啥了

说白了就是由软件自身调度的线程，传统的线程是由操作系统调度的，是操作系统线程的简单粗暴的包装，那传统操作系统线程的问题就是

内存占用大，一个线程一般都需要1m的栈，堆的部分可以共享

然后传统的io密集型应用的写法，就是每一个请求-响应，就会占用一个线程

那如果请求比较多，一次就要弄掉1m，很快内存就会耗尽

那虚拟线程占用的内存只有十几个字节，在io的时候，它可以把阻塞的线程，改为虚拟线程，并把正在执行虚拟线程的操作系统线程（也就是carrier thread，执行线程）腾出来，去做其他操作，这样多次请求就能共享操作系统线程了

那对于io密集型应用而言，io的吞吐就大幅增加了

其实大部分所谓的业务系统后端，都是crud，也就是io密集型应用，虚拟线程这个功能，对于这类系统性能的提升，是非常明显的

第二个这次虚拟线程，它自带了stack，当然这个stack也是虚拟的，jvm会帮你调节这个stack的大小，因为是有栈的，所以它的适用范围更广，简单说就是可以完全模拟线程



作者：圆胖肿
链接：https://www.zhihu.com/question/536743167/answer/2664084546
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。