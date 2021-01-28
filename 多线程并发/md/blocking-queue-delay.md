 

## DelayQueue概述

DelayQueue是一个**支持延时获取元素**的无界阻塞队列，使用PriorityQueue来存储元素。

队中的元素必须实现`Delayed`接口【Delay接口又继承了Comparable，需要实现compareTo方法】，每个元素都需要指明过期时间，通过`getDelay(unit)`获取元素剩余时间【剩余时间 = 到期时间 - 当前时间】，每次向优先队列中添加元素时根据compareTo方法作为排序规则。

当从队列获取元素时，只有过期的元素才会出队列。

## 类图及重要字段

![image-20210128220342776](img/blocking-queue-delay/image-20210128220342776.png)

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> {
    // 独占锁实现同步
    private final transient ReentrantLock lock = new ReentrantLock();
    // 优先队列存放数据
    private final PriorityQueue<E> q = new PriorityQueue<E>();

    /**
     * Thread designated to wait for the element at the head of
     * the queue.  This variant of the Leader-Follower pattern
     * (http://www.cs.wustl.edu/~schmidt/POSA/POSA2/) serves to
     * minimize unnecessary timed waiting.  When a thread becomes
     * the leader, it waits only for the next delay to elapse, but
     * other threads await indefinitely.  The leader thread must
     * signal some other thread before returning from take() or
     * poll(...), unless some other thread becomes leader in the
     * interim.  Whenever the head of the queue is replaced with
     * an element with an earlier expiration time, the leader
     * field is invalidated by being reset to null, and some
     * waiting thread, but not necessarily the current leader, is
     * signalled.  So waiting threads must be prepared to acquire
     * and lose leadership while waiting.
     * 基于Leader-Follower模式的变体,用于尽量减少不必要的线程等待
     */
    private Thread leader = null;

    /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
     * 与lock对应的条件变量
     */
    private final Condition available = lock.newCondition();    
}
```

## Leader-Follower模式





## Delayed接口

队中的元素必须实现`Delayed`接口【Delay接口又继承了Comparable，需要实现compareTo方法】，每个元素都需要指明过期时间，通过`getDelay(unit)`获取元素剩余时间【剩余时间 = 到期时间 - 当前时间】。

每次向优先队列中添加元素时根据compareTo方法作为排序规则，当然我们约定一下，默认q.peek()出来的就是最先过期的元素。

```java
public interface Delayed extends Comparable<Delayed> {
    // 返回剩余时间
    long getDelay(TimeUnit unit);
}

public interface Comparable<T> {
	// 定义比较方法
    public int compareTo(T o);
}
```

## 构造器

DelayQueue构造器相比于前几个，就显得非常easy了。

```java
    public DelayQueue() {}

    public DelayQueue(Collection<? extends E> c) {
        this.addAll(c);
    }
```

## put

因为DelayQueue是无界队列，不会因为边界问题产生阻塞，因此put操作和offer操作是一样的。

```java
    public void put(E e) {
        offer(e);
    }

    public boolean offer(E e) {
        // 获取独占锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 加入优先队列里
            q.offer(e);
            // 判断堆顶元素是不是刚刚插入的元素
            // 如果判断为true，说明当前这个元素是将最先过期
            if (q.peek() == e) {
                // 重置leader线程为null
                leader = null; 
                // 激活available变量条件队列中的一个线程
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
```

## take

take方法将会**获取并移除队列里面延迟时间过期的元素** ，如果队列里面没有过期元素则陷入等待。

```java
    public E take() throws InterruptedException {
        // 获取独占锁
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                // 瞅一瞅谁最快过期
                E first = q.peek();
                // 队列为空，则将当前线程置入available的条件队列中，直到里面有元素
                if (first == null)
                    available.await();
                else {
                    // 看下还有多久过期
                    long delay = first.getDelay(NANOSECONDS);
                    // 哇，已经过期了，就移除它并返回
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    // leader不为null表示其他线程也在执行take
                    // 则将当前线程置入available的条件队列中
                    if (leader != null)
                        available.await();
                    else {
                        // 如果leader为null，则选择当前线程作为leader线程
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            // 等待delay时间，时间到之后，会出条件队列，继续竞争锁
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```

### first = null 有什么用

如果不设置`first = null`，将会引起内存泄露。

> - 线程A到达，队首元素没有到期，设置leader = 线程A，并且执行`available.awaitNanos(delay);`等待元素过期。
> - 这时线程B来了，因为leader != null，则会`available.await();`阻塞，线程C、D、E同理。
> - 线程A阻塞完毕了，再次循环，获取列首元素成功，出列。
>
> 这个时候列首元素应该会被回收掉，但是问题是它还被线程B、线程C持有着，所以不会回收，如果线程增多，且队首元素无限期的不能回收，就会造成内存泄漏。

## 总结

DelayQueue是一个**支持延时获取元素**的**无界阻塞**队列，使用PriorityQueue来存储元素。

队中的元素必须实现`Delayed`接口【Delay接口又继承了Comparable，需要实现compareTo方法】，每个元素都需要指明过期时间，通过`getDelay(unit)`获取元素剩余时间【剩余时间 = 到期时间 - 当前时间】，每次向优先队列中添加元素时根据compareTo方法作为排序规则。

基于Leader-Follower模式使用leader变量，减少不必要的线程等待。

DelayQueue是无界队列，因此插入操作是非阻塞的。但是take操作从队列获取元素时，是阻塞的，阻塞规则为：



## 参考阅读

- 《Java并发编程的艺术》
- 《Java并发编程之美》