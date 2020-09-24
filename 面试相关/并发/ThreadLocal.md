# ThreadLocal是啥？

ThreadLocal类顾名思义可以理解为线程本地变量。也就是说如果定义了一个ThreadLocal，每个线程往这个ThreadLocal中读写是线程隔离，互相之间不会影响的。它提供了一种将可变数据通过每个线程有自己的独立副本从而实现线程封闭的机制。

# ThreadLocal的实现思路？

- Thread类中有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，意味着每个线程都有一个自己的ThreadLocalMap。
- 可以简单地将key视作ThreadLocal，value为代码中放入的值（实际上key并不是ThreadLocal本身，而是它的一个弱引用）。
- 每个线程在往某个ThreadLocal里塞值的时候，都会往自己的ThreadLocalMap里存，读也是以某个ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。



# ThreadLocalMap源码分析

## 存储结构



## 为啥要用弱引用	

# ThreadLocal与内存泄漏

ThreadLocal具体存放变量的是线程的threadLocals变量，threadLocals是一个ThreadLocalMap类型的变量，内部是一个Entry数组，Entry继承自WeakReference,Entry内部的value用来存放通过ThreadLocal的set方法传递的值。

key是ThreadLocal的弱引用，key虽然会被GC回收，但value不能被回收，这时候ThreadLocalMap中会存在key为null，value不为null的entry项，如果时间长了就会存在大量无用对象，造成OOM。

虽然set,get也提供了一些对Entry项清理的时机，但不及时，所以在使用完毕后需要及时调用remove。



- [https://www.cnblogs.com/micrari/p/6790229.html](https://www.cnblogs.com/micrari/p/6790229.html)