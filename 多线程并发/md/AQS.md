[toc]

> 本文基于JDK1.8

## 本篇学习目标

- 了解AQS的设计思想以及重要字段含义，如通过state字段表示同步状态等。
- 了解AQS内部维护链式双向同步队列的结构以及几个重要指针。
- 了解五种重要的同步状态。
- 明确两种模式：共享模式和独占模式。
- 学习两种模式下AQS提供的模板方法：获取与释放同步状态相关方法。
- 了解Condition、ConditionObject等AQS对条件变量的支持。
- 通过Condition的案例深入了解等待通知的机制。

## AQS概述

AQS即`AbstractQueuedSynchronizer`，队列同步器，他是构建众多同步组件的基础框架，如`ReentrantLock`、`ReentrantReadWriteLock`等，是J.U.C并发包的核心基础组件。

AQS框架基于模板方法设计模式构建，子类通过继承它，实现它提供的抽象方法来管理同步状态。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
	// 序列化版本号
    private static final long serialVersionUID = 7373984972572414691L;

    /**
     * 创建AQS实例，初始化state为0，为子类提供
     */
    protected AbstractQueuedSynchronizer() { }
    
    /*----------------  同步队列构成 ---------------*/

    // 等待队列节点类型
    static final class Node {
        //...省略
    }

    /**
     * 除了初始化之外，它只能通过setHead方法进行修改。注意:如果head存在，它的waitStatus保证不会被取消
     */
    private transient volatile Node head;

    /**
     * 等待队列的尾部，懒初始化，之后只在enq方法加入新节点时修改
     */
    private transient volatile Node tail;
    
    /*----------------  同步状态相关 ---------------*/

    /**
     * volatile修饰， 标识同步状态，state为0表示锁空闲，state>0表示锁被持有，可以大于1，表示被重入
     */
    private volatile int state;

    /**
     * 返回当前同步状态
     */
    protected final int getState() {
        return state;
    }

    /**
     * 设置同步状态
     */
    protected final void setState(int newState) {
        state = newState;
    }

    /**
     * 利用CAS操作更新state值
     */
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    static final long spinForTimeoutThreshold = 1000L;
    
    // 这部分和CAS有关 
    // 获取Unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // 记录state在AQS类中的偏移值
    private static final long stateOffset;
    static {
        try {
            // 初始化state变量的偏移值
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        } catch (Exception ex) { throw new Error(ex); }
    }
}
```

以上包括AQS的基本字段，比较核心的就是两个部分：

- 内部FIFO队列的节点类型Node，和首尾指针字段。
- 同步状态相关的方法，设置，获取，CAS更新等。

接下来我们将一一学习这些内容。

## AbstractOwnableSynchronizer

AbstractQueuedSynchronizer继承自AbstractOwnableSynchronizer，它提供了**设置或获取独占锁的拥有者线程**的功能。

```java
public abstract class AbstractOwnableSynchronizer
    implements java.io.Serializable {
 
    private static final long serialVersionUID = 3737899427754241961L;
 	// 本身是抽象类，该构造方法实为为子类提供
    protected AbstractOwnableSynchronizer() { }
 
	/* 互斥模式同步下的当前线程 */
    private transient Thread exclusiveOwnerThread;
 
	/* 设置当前拥有独占访问的线程。锁的拥有线程，null 参数表示没有线程拥有访问。
     * 此方法不另外施加任何同步或 volatile 字段访问。
     */
    protected final void setExclusiveOwnerThread(Thread t) {
        exclusiveOwnerThread = t;
    }
 
	/* 返回由 setExclusiveOwnerThread 最后设置的线程；
     * 如果从未设置，则返回 null。
     * 此方法不另外施加任何同步或 volatile 字段访问。 
     */
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```

这里exclusiveOwnerThread字段用来判断当前线程是不是持有锁，因为锁可以重入嘛，因此就会产生下面这样的伪代码：

```java
if (currThread == getExclusiveOwnerThread()) {
	state++;
}
```

## 同步队列与Node节点

> tips：同步队列被称为CLH队列，是Craig，Landin，Hagersten的合称。

AQS通过内置的FIFO同步双向队列来完成资源获取线程的排队工作，内部通过节点head【实际上是虚拟节点，真正的第一个线程在head.next的位置】和tail记录队首和队尾元素，队列元素类型为Node。

CLU同步队列的结构如下，具体的操作之后再做总结：

> 图源：[【AQS】核心实现](http://cmsblogs.com/?p=11021)

![](img/AQS/1771072-20210105165819676-1131360236.png)


- 如果当前线程获取同步状态失败（锁）时，AQS 则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程
- 当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。

```java
    /**
     * 等待队列中的节点类
     */
    static final class Node {
        /** 标识共享式节点 */
        static final Node SHARED = new Node();
        /** 标识独占式节点 */
        static final Node EXCLUSIVE = null;

        /** ------------ 等待状态 ---------------*/
        
        /** 表示该线程放弃对锁的争夺 */
        static final int CANCELLED =  1;
        /** 当前node的后继节点对应的线程需要被唤醒 */
        static final int SIGNAL    = -1;
        /** 线程在条件队列中等待 */
        static final int CONDITION = -2;
        /** 释放共享资源时需要通知其他节点 */
        static final int PROPAGATE = -3;
        
        /** waitStatus == 0 表示不是以上任何一种 */
        
        // 记录当前线程的等待状态，以上五种
        volatile int waitStatus;

        // 前驱节点
        volatile Node prev;

        // 后继节点
        volatile Node next;

        // node存储的线程
        volatile Thread thread;
		
        // 当前节点在Condition中等待队列上的下一个节点
        Node nextWaiter;

        // 判断是否为共享式获取同步状态
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * 为什么不直接判断prev，而是用p变量判断呢？
         * 避免并发的情况下，prev判断完为null，恰好被修改
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
		// 用于SHARED的创建
        Node() {    
        }
		// 用于addWaiter(Node mode)方法，指定模式的
        Node(Thread thread, Node mode) {     
            this.nextWaiter = mode;
            this.thread = thread;
        }
		// 用于addConditionWaiter()方法
        Node(Thread thread, int waitStatus) { 
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

注释中已经标注各个等待状态的取值和表示，这边再总结一下：

- 等待状态使用waitStatus字段表示，用来控制线程的阻塞和唤醒，除了上面写的四种，实际还有一种状态是0，这在源码的注释中已经明确。
- 【CANCELLED = 1】取消状态，由于超时或中断，节点会被设置为取消状态，且一直保持，不会参与到竞争中。如果waitStatus>0则可以认为是该线程取消了等待。
- 【SIGNAL = -1】后继节点的线程处于等待状态，当前节点的线程如果释放了同步状态或被取消，将通知后继节点，使后继节点的线程得以运行。
- 【CONDITION = -2】节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中。
- 【PROPAGATE = -3】表示下一次共享式同步状态获取，将会无条件地传播下去。

- 0：初始状态，上面几种啥也不是。

> 关于AQS内部维护的同步队列，这里只是了解一些基本概念，后续对队列操作注意点进行深入学习。

## 同步状态state

AQS使用一个int类型的成员变量state来表示同步状态，它用volatile修饰，并且提供了关于state的三个线程安全的方法：

- `getState()`，读取同步状态。
- `setState(int newState)`，设置同步状态为newState。
- `compareAndSetState(int expect, int update)`，CAS操作更新state值。

state为0表示锁空闲，state>0表示锁被持有，可以大于1，表示被重入。不同的子类实现，对state的含义表示略有差异，举几个例子吧：

- ReentrantLock：state表示当前线程获取锁的可重入次数。
- ReetrantReadWriteLock：state的高16位表示读状态，也就是获取该读锁的次数，低16位表示获取到写锁的线程的可重入次数。
- semaphore：state表示当前可用信号的个数。
- CountDownlatch：state表示计数器当前的值。

## 重要方法分析

对于AQS来说，线程同步的关键是对state进行操作，根据state是否属于一个线程，操作state的方式分为独占方式和共享方式。

### 独占式获取与释放同步状态

- 使用独占的方式获取的资源是与具体线程绑定的，如果一个线程获取到了资源，便标记这个线程已经获取到，其他线程再次尝试操作state获取资源时就会发现当前该资源不是自己持有的，就会在获取失败后阻塞。

> 比如独占锁ReentrantLock的实现，当一个线程获取了ReentrantLock的锁后，在AQS内部会首先使用CAS操作把state状态值从0变为1，然后设置当前锁的持有者为当前线程，当该线程再次获取锁时发现它就是锁的持有者，则会把状态值从1变为2，也就是设置可重入次数，而当另外一个线程获取锁时发现自己并不是该锁的持有者就会被放入AQS阻塞队列后挂起。

```java
	// 独占式获取同步状态，成功后，其他线程需要等待该线程释放同步状态才能获取同步状态
    public final void acquire(int arg) {
        // 首先调用 tryAcquire【需要子类实现】尝试获取资源，本质就是设置state的值，获取成功就直接返回
        if (!tryAcquire(arg) &&
            // 获取失败，就将当前线程封装成类型为Node.EXCLUSIVE的Node节点，并插入AQS阻塞队列尾部
            // 然后通过自旋获取同步状态
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

	// 与 acquire(int arg) 相同，但是该方法响应中断。
	// 如果其他线程调用了当前线程的interrupt()方法，响应中断，抛出异常。
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        // interrupted()方法将会获取当前线程的中断标志并重置
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
	//尝试获取锁，如果获取失败会将当前线程挂起指定时间，时间到了之后当前线程被激活，如果还是没有获取到锁，就返回false。
	//另外，该方法会对中断进行的响应，如果其他线程调用了当前线程的interrupt()方法，响应中断，抛出异常。
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }
	// 独占式释放同步状态
    public final boolean release(int arg) {
        // 尝试使用tryRelease释放资源，本质也是设置state的值
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // LockSupport.unpark(thread) 激活AQS里面被阻塞的一个线程
                // 被激活的线程则使用tryAcquire 尝试，看当前状态变量state的值是否能满足自己的需要，
                //满足则该线程被激活，然后继续向下运行，否则还是会被放入AQS队列并被挂起。
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

需要注意：tryRelease和tryAcquire方法并没有在AQS中给出实现，实现的任务交给了具体的子类，子类根据具体的场景需求实现，通过CAS算法，设置修改state的值。

### 共享式获取与释放同步状态

- 对应共享方式的资源与具体线程是不相关的，当多个线程去请求资源时通过CAS 方式竞争获取资源，当一个线程获取到了资源后，另外一个线程再次去获取时如果当前资源还能满足它的需要，则当前线程只需要使用CAS 方式进行获取即可。

> 比如Semaphore信号量，当一个线程通过acquire()方法获取信号量时，会首先看当前信号量个数是否满足需要，不满足则把当前线程放入阻塞队列，如果满足则通过自旋CAS获取信号量。

```java
    //共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，
	// 与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
	public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            // 尝试获取资源，如果成功则直接返回
            // 如果失败，则将当前线程封装为类型为Node.SHARED的Node节点并插入AQS阻塞队列尾部
            // 并使用LockSupport.park(this)挂起自己
            doAcquireShared(arg);
    }
	// 共享式获取同步状态，响应中断
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
	//共享式获取同步状态，增加超时限制
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
            doAcquireSharedNanos(arg, nanosTimeout);
    }
	//共享式释放同步状态
    public final boolean releaseShared(int arg) {
        // 尝试释放资源
        if (tryReleaseShared(arg)) {
            // 调用LockSupport.unpark(thread)激活AQS队列里被阻塞的一个线程。
            // 被激活的线程使用tryReleaseShared查看当前状态变量state是否能满足自己的需要。
            // 如果满足需要，则线程被激活继续向下运行，否则还是放入AQS队列并被挂起
            doReleaseShared();
            return true;
        }
        return false;
    }
```

Interruptibly的方法表示对中断需要进行响应，线程在调用带Interruptibly关键字的方法获取资源时或者获取资源失败被挂起，其他线程中断了该线程，那么该线程会抛出`InterruptedException`异常而返回。

## AQS条件变量的支持

### Condition接口

Contition是一种广义上的条件队列，它利用await()和signal()为线程提供了一种**更为灵活的等待/通知模式**。

Condition必须要配合Lock一起使用，因为对共享状态变量的访问发生在多线程环境下。一个Condition的实例必须与一个Lock绑定，因此await和signal的调用必须在lock和unlock之间，有锁之后，才能使用condition嘛。以ReentrantLock为例，简单使用如下：

```java
public class ConditionTest {

    public static void main(String[] args) {
        final ReentrantLock lock = new ReentrantLock();
        final Condition condition = lock.newCondition();

        Thread thread1 = new Thread(() -> {
            String name = Thread.currentThread().getName();

            lock.lock();
            System.out.println(name + " <==成功获取到锁" + lock);
            try {
                System.out.println(name + " <==进入条件队列等待");
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(name + " <==醒了");
            lock.unlock();
            System.out.println(name + " <==释放锁");
        }, "等待线程");

        thread1.start();

        Thread thread2 = new Thread(() -> {
            String name = Thread.currentThread().getName();

            lock.lock();
            System.out.println(name + " ==>成功获取到锁" + lock);
            try {
                System.out.println("========== 这里演示await中的线程没有被signal的时候会一直等着 ===========");
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(name + " ==>通知等待队列的线程");
            condition.signal();
            lock.unlock();
            System.out.println(name + " ==>释放锁");
        }, "通知线程");

        thread2.start();
    }
}
```

```java
等待线程 <==成功获取到锁java.util.concurrent.locks.ReentrantLock@3642cea8[Locked by thread 等待线程]
等待线程 <==进入条件队列等待
通知线程 ==>成功获取到锁java.util.concurrent.locks.ReentrantLock@3642cea8[Locked by thread 通知线程]
========== 这里演示await中的线程没有被signal的时候会一直等着 ===========
通知线程 ==>通知等待队列的线程
通知线程 ==>释放锁
等待线程 <==醒了
等待线程  <==释放锁
```

### ConditionObject内部类

`AQS，Lock,Condition，ConditionObject`之间的关系：

ConditionObject是AQS的内部类，实现了Condition接口，Lock中提供newCondition()方法，委托给内部AQS的实现Sync来创建ConditionObject对象，享受AQS对Condition的支持。

```java
    // ReentrantLock#newCondition
	public Condition newCondition() {
        return sync.newCondition();
    }
	// Sync#newCondition
    final ConditionObject newCondition() {
        // 返回Contition的实现，定义在AQS中
        return new ConditionObject();
    }
```

ConditionObject用来结合锁实现线程同步，**ConditionObject可以直接访问AQS对象内部的变量，比如state状态值和AQS队列**。ConditionObject是条件变量，每个条件变量对应一个**条件队列**（单向链表队列），其用来存放调用条件变量的await方法后被阻塞的线程，ConditionObject维护了首尾节点：

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;
}
```

看到这里我们需要明确这里的条件队列和上面分析的同步队列是不一样的：

- AQS维护的是当前在等待资源的队列，Condition维护的是在等待signal信号的队列。

- 每个线程会存在上述两个队列中的一个，lock与unlock对应在AQS队列，signal与await对应条件队列，线程节点在他们之间反复横跳。

这里我们针对上面的demo来分析一下会更好理解一些：

> 为了简化，接下来我将**用D表示等待线程，用T表示通知线程**。

1. 【D】先调用`lock.lock()`方法，此时无竞争，【D】被加入到AQS等待队列中。
2. 【D】调用`condition.await()`方法，此时【D】从AQS等待队列中移除，并加入到condition对应的条件等待队列中。
3. 【D】陷入等待之后，【T】启动，由于AQS队列中的【D】已经被移除，此时【T】也很快获取到锁，相应的，【T】也被加入到AQS等待队列中。
4. 【T】接着调用`condition.signal()`方法，这时condition对应的条件队列中只有一个节点【D】，于是【D】被取出，并被再次加入AQS的等待队列中。此时【D】并没有被唤醒，只是单纯换了个位置。
5. 接着【T】执行`lock.unlock()`，释放锁锁之后，会唤醒AQS队列中的【D】，此时【D】真正被唤醒且执行。

你看，【D】线程确实在两个队列中反复横跳吧，有关Condition的内容本文也只是抛砖引玉，之后会作详细学习总结，如果你对这一系列感兴趣，可以关注一下，后续会陆续对并发进行深入学习。

## 参考阅读

- [【死磕Java并发】—–J.U.C之AQS：AQS简介](http://cmsblogs.com/?p=2174)
- [【死磕Java并发】—–J.U.C之Condition](http://cmsblogs.com/?p=2222)
- [怎么理解Condition](http://cmsblogs.com/?p=2878)
- [一行一行源码分析清楚AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)
- 《并发编程之美》