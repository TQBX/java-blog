[toc]

## Condition接口

Contition是一种广义上的条件队列，它利用await()和signal()为线程提供了一种**更为灵活的等待/通知模式**。

Condition必须要配合Lock一起使用，因为对共享状态变量的访问发生在多线程环境下。

**一个Condition的实例必须与一个Lock绑定，因此await和signal的调用必须在lock和unlock之间**，**有锁之后，才能使用condition**嘛。以ReentrantLock为例，简单使用如下：

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
等待线程 <==成功获取到锁java.util.concurrent.locks.ReentrantLock@3642cea8[Locked by thread 等待线程]
等待线程 <==进入条件队列等待
通知线程 ==>成功获取到锁java.util.concurrent.locks.ReentrantLock@3642cea8[Locked by thread 通知线程]
========== 这里演示await中的线程没有被signal的时候会一直等着 ===========
通知线程 ==>通知等待队列的线程
通知线程 ==>释放锁
等待线程 <==醒了
等待线程  <==释放锁
```

接下来我们将从源码的角度分析上面这个流程，理解所谓条件队列的内涵。

## AQS条件变量的支持之ConditionObject内部类

`AQS，Lock,Condition，ConditionObject`之间的关系：

**ConditionObject是AQS的内部类，实现了Condition接口**，Lock中提供newCondition()方法，委托给内部AQS的实现Sync来创建ConditionObject对象，享受AQS对Condition的支持。

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

ConditionObject用来结合锁实现线程同步，**ConditionObject可以直接访问AQS对象内部的变量，比如state状态值和AQS队列**。

ConditionObject是条件变量，每个条件变量对应一个**条件队列**（单向链表队列），其用来存放调用条件变量的await方法后被阻塞的线程，ConditionObject维护了首尾节点，没错这里的Node就是我们之前在学习AQS的时候见到的那个Node，我们会在下面回顾：

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** 条件队列的第一个节点. */
    private transient Node firstWaiter;
    /** 条件队列的最后一个节点. */
    private transient Node lastWaiter;
}
```

看到这里我们需要明确**这里的条件队列和我们之前说的AQS同步队列是不一样**的：

- AQS维护的是当前在等待资源的队列，Condition维护的是在等待signal信号的队列。
- 每个线程会存在上述两个队列中的一个，lock与unlock对应在AQS队列，signal与await对应条件队列，线程节点在他们之间反复横跳。

这里我们针对上面的demo来分析一下会更好理解一些：

> 为了简化，接下来我将**用D表示等待线程，用T表示通知线程**。

1. 【D】先调用`lock.lock()`方法，此时无竞争，【D】被加入到AQS等待队列中。
2. 【D】调用`condition.await()`方法，此时【D】从AQS等待队列中移除，并加入到condition对应的条件等待队列中。
3. 【D】陷入等待之后，【T】启动，由于AQS队列中的【D】已经被移除，此时【T】也很快获取到锁，相应的，【T】也被加入到AQS等待队列中。
4. 【T】接着调用`condition.signal()`方法，这时condition对应的条件队列中只有一个节点【D】，于是【D】被取出，并被再次加入AQS的等待队列中。此时【D】并没有被唤醒，只是单纯换了个位置。
5. 接着【T】执行`lock.unlock()`，释放锁锁之后，会唤醒AQS队列中的【D】，此时【D】真正被唤醒且执行。

## 回顾AQS中的Node

我们这里再简单回顾一下AQS中Node类与Condition相关的字段：

```java
        // 记录当前线程的等待状态，
        volatile int waitStatus;

        // 前驱节点
        volatile Node prev;

        // 后继节点
        volatile Node next;

        // node存储的线程
        volatile Thread thread;
		
        // 当前节点在Condition中等待队列上的下一个节点
        Node nextWaiter;
```

waitStatus可以取五种状态：

1. 初始化为0，啥也不表示，之后会被置signal。
2. 1表示cancelled，取消当前线程对锁的争夺。
3. -1表示signal，表示当前节点释放锁后需要唤醒后面可被唤醒的节点。
4. -2表示condition，我们这篇的重点，**表示当前节点在条件队列中**。
5. -3表示propagate，表示释放共享资源的时候会向后传播释放其他共享节点。

当然，除了-2这个condition状态，其他的等待状态我们之前都或多或少分析过，今天着重学习condition这个状态的意义。

我们还可以看到一个Node类型的nextWaiter，它表示**条件队列中当前节点的下一个节点**，可以看出用以实现条件队列的单向链表。

## await()

了解这些之后，我们直接来看看具体方法的源码：

