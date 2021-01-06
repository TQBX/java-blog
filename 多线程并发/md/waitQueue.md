[toc]

## 本篇学习目标

- 回顾CLH同步队列的结构。
- 学习独占式资源获取和释放的流程。

## CLH队列的结构

我在[Java并发包源码学习系列：AbstractQueuedSynchronizer#同步队列与Node节点](https://www.cnblogs.com/summerday152/p/14238284.html#同步队列与node节点)已经粗略地介绍了一下CLH的结构，本篇主要解析该同步队列的相关操作，因此在这边再回顾一下：

AQS通过内置的FIFO同步双向队列来完成资源获取线程的排队工作，内部通过节点head【实际上是虚拟节点，真正的第一个线程在head.next的位置】和tail记录队首和队尾元素，队列元素类型为Node。

![AQS 同步FIFO双向链式队列结构](img/waitQueue/1771072-20210105230806800-1493816267.png)

- 如果当前线程获取同步状态失败（锁）时，AQS 则会将当前线程以及等待状态等信息构造成一个节点（Node）并将其加入同步队列，同时会阻塞当前线程
- 当同步状态释放时，则会把节点中的线程唤醒，使其再次尝试获取同步状态。

> 接下来将要通过AQS以独占式的获取和释放资源的具体案例来详解内置CLU阻塞队列的工作流程，接着往下看吧。

## 资源获取

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && // tryAcquire由子类实现，表示获取锁，如果成功，这个方法直接返回了
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 如果获取失败，执行
            selfInterrupt();
    }
```

- tryAcquire(int)是AQS提供给子类实现的钩子方法，子类可以自定义实现独占式获取资源的方式，获取成功则返回true，失败则返回false。
- 如果tryAcquire方法获取资源成功就直接返回了，失败的化就会执行`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`的逻辑，我们可以将其进行拆分，分为两步：
  - addWaiter(Node.EXCLUSIVE)：将该线程包装成为独占式的节点，加入队列中。
  - acquireQueued(node,arg)：如果当前节点是等待节点的第一个，即head.next，就尝试获取资源。如果该方法返回true，则会进入`selfInterrupt()`的逻辑，进行阻塞。

接下来我们分别来看看`addWaiter`和`acquireQueued`两个方法。

### 入队Node addWaiter(Node mode)

根据传入的mode参数决定独占或共享模式，为当前线程创建节点，并入队。

```java
    // 其实就是把当前线程包装一下，设置模式，形成节点，加入队列
	private Node addWaiter(Node mode) {
        // 根据mode和thread创建节点
        Node node = new Node(Thread.currentThread(), mode);
        // 记录一下原尾节点
        Node pred = tail;
        // 尾节点不为null，队列不为空，快速尝试加入队尾。
        if (pred != null) {
            // 让node的prev指向尾节点
            node.prev = pred;
            // CAS操作设置node为新的尾节点，tail = node
            if (compareAndSetTail(pred, node)) {
                // 设置成功，让原尾节点的next指向新的node，实现双向链接
                pred.next = node;
                // 入队成功，返回
                return node;
            }
        }
        // 快速入队失败，进行不断尝试
        enq(node);
        return node;
    }
```

几个注意点：

- 入队的操作其实就是将线程通过指定模式包装为Node节点，如果队列尾节点不为null，利用CAS尝试快速加入队尾。
- 快速入队失败的原因有两个：
  - 队列为空，即还没有进行初始化。
  - CAS设置尾节点的时候失败。
- 在第一次快速入队失败后，将会走到enq(node)逻辑，不断进行尝试，直到设置成功。

### 不断尝试Node enq(final Node node)

```java
    private Node enq(final Node node) {
        // 自旋，俗称死循环，直到设置成功为止
        for (;;) {
            // 记录原尾节点
            Node t = tail;
            // 第一种情况：队列为空，原先head和tail都为null，
            // 通过CAS设置head为哨兵节点，如果设置成功，tail也指向哨兵节点
            if (t == null) { // Must initialize
                // 初始化head节点
                if (compareAndSetHead(new Node()))
                    // tail指向head，下个线程来的时候，tail就不为null了，就走到了else分支
                    tail = head;
            // 第二种情况：CAS设置尾节点失败的情况，和addWaiter一样，只不过它在for(;;)中
            } else {
                // 入队，将新节点的prev指向tail
                node.prev = t;
                // CAS设置node为尾部节点
                if (compareAndSetTail(t, node)) {
                    //原来的tail的next指向node
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

enq的过程是自选设置队尾的过程，如果设置成功，就返回。如果设置失败，则一直尝试设置，理念就是，我总能等待设置成功那一天。

我们还可以发现，head是延迟初始化的，在第一个节点尝试入队的时候，head为null，这时使用了`new Node()`创建了一个不代表任何线程的节点，作为虚拟头节点，且我们需要注意它的waitStatus初始化为0，这一点对我们之后分析有指导意义。

如果是CAS失败导致重复尝试，那就还是让他继续CAS好了。

### boolean acquireQueued(Node, int)

```java
    // 这个方法如果返回true，代码将进入selfInterrupt()
	final boolean acquireQueued(final Node node, int arg) {
        // 注意默认为true
        boolean failed = true;
        try {
            // 是否中断
            boolean interrupted = false;
            // 自旋，即死循环
            for (;;) {
                // 得到node的前驱节点
                final Node p = node.predecessor();
                // 我们知道head是虚拟的头节点，p==head表示如果node为阻塞队列的第一个真实节点
                // 就执行tryAcquire逻辑，这里tryAcquire也需要由子类实现
                if (p == head && tryAcquire(arg)) {
                    // tryAcquire获取成功走到这，执行setHead出队操作 
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 走到这有两种情况 1.node不是第一个节点 2.tryAcquire争夺锁失败了
                // 这里就判断 如果当前线程争锁失败，是否需要挂起当前这个线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            // 死循环退出，只有tryAcquire获取锁失败的时候failed才为true
            if (failed)
                cancelAcquire(node);
        }
    }
```

### 出队void setHead(Node)

CLU同步队列遵循FIFO，首节点的线程释放同步状态后，唤醒下一个节点。将队首节点出队的操作实际上就是，将head指针指向将要出队的节点就可以了。

```java
    private void setHead(Node node) {
        // head指针指向node
        head = node;
        // 释放资源
        node.thread = null;
        node.prev = null;
    }
```

### boolean shouldParkAfterFailedAcquire(Node,Node)

```java
    /**
     * 走到这有两种情况 1.node不是第一个节点 2.tryAcquire争夺锁失败了
     * 这里就判断 如果当前线程争锁失败，是否需要挂起当前这个线程
     *
     * 这里pred是前驱节点， node就是当前节点
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 前驱节点的waitStatus
        int ws = pred.waitStatus;
        // 前驱节点为SIGNAL【-1】直接返回true，表示当前节点可以被直接挂起
        if (ws == Node.SIGNAL)
            return true;
        // ws>0 CANCEL 说明前驱节点取消了排队
        if (ws > 0) {
            // 下面这段循环其实就是跳过所有取消的节点，找到第一个正常的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            // 将该节点的后继指向node，建立双向连接
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             * 官方说明：走到这waitStatus只能是0或propagate，默认情况下，当有新节点入队时，waitStatus总是为0
             * 下面用CAS操作将前驱节点的waitStatus值设置为signal
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 返回false，接着会再进入循环，此时前驱节点为signal，返回true
        return false;
    }
```

针对前驱节点的waitStatus有三种情况：

> 等待状态不会为 `Node.CONDITION` ，因为它用在 ConditonObject 中

1. ws==-1，即为Node.SIGNAL,表示当前节点node可以被直接挂起，在pred线程释放同步状态时，会对node线程进行唤醒。
2. ws > 0，即为Node.CANCELLED，说明前驱节点已经取消了排队【可能是超时，可能是被中断】，则需要找到前面没有取消的前驱节点，一直找，直到找到为止。
3. ws == 0 or ws == Node.PROPAGATE：
   - 默认情况下，当有新节点入队时，waitStatus总是为0，用CAS操作将前驱节点的waitStatus值设置为signal，下一次进来的时候，就走到了第一个分支。
   - 当释放锁的时候，会将占用锁的节点的ws状态更新为0。

> PROPAGATE表示共享模式下，前驱节点不仅会唤醒后继节点，同时也可能会唤醒后继的后继。

我们可以发现，这个方法在第一次走进来的时候是不会返回true的。原因在于，返回true的条件时前驱节点的状态为SIGNAL，而第一次的时候还没有给前驱节点设置SIGNAL呢，只有在CAS设置了状态之后，第二次进来才会返回true。

那SIGNAL的意义到底是什么呢？

>这里引用：[并发编程——详解 AQS CLH 锁 # 为什么 AQS 需要一个虚拟 head 节点](https://www.jianshu.com/p/4682a6b0802d)
>
>waitStatus这里用ws简称，每个节点都有ws变量，用于表示该节点的状态。初始化的时候为0，如果被取消为1，signal为-1。
>
>如果某个节点的状态是signal的，那么在该节点释放锁的时候，它需要唤醒下一个节点。
>
>因此，每个节点在休眠之前，如果没有将前驱节点的ws设置为signal，那么它将永远无法被唤醒。
>
>因此我们会发现上面当前驱节点的ws为0或propagate的时候，采用cas操作将ws设置为signal，目的就是让上一个节点释放锁的时候能够通知自己。

### boolean parkAndCheckInterrupt()

```java
    private final boolean parkAndCheckInterrupt() {
        // 挂起当前线程
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

shouldParkAfterFailedAcquire方法返回true之后，就会调用该方法，挂起当前线程。

`LockSupport.park(this)`方法挂起的线程有两种途径被唤醒：1.被unpark() 2.被interrupt()。

需要注意这里的Thread.interrupted()会清除中断标记位。

### void cancelAcquire(node)

上面tryAcquire获取锁失败的时候，会走到这个方法。

```java
    private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;
		// 将节点的线程置空
        node.thread = null;

        // 跳过所有的取消的节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        // 这里在没有并发的情况下，preNext和node是一致的
        Node predNext = pred.next;

        // Can use unconditional write instead of CAS here. 可以直接写而不是用CAS
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        // 设置node节点为取消状态
        node.waitStatus = Node.CANCELLED;

        // 如果node为尾节点就CAS将pred设置为新尾节点
        if (node == tail && compareAndSetTail(node, pred)) {
            // 设置成功之后，CAS将pred的下一个节点置为空
            compareAndSetNext(pred, predNext, null);
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head && // pred不是首节点
                ((ws = pred.waitStatus) == Node.SIGNAL || // pred的ws为SIGNAL 或 可以被CAS设置为SIGNAL
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) { // pred线程非空
                // 保存node 的下一个节点
                Node next = node.next; 
                // node的下一个节点不是cancelled，就cas设置pred的下一个节点为next
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);
            } else {
                // 上面的情况除外，则走到这个分支，唤醒node的下一个可唤醒节点线程
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }
```

## 释放资源

### boolean release(int arg)

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) { // 子类实现tryRelease方法
            // 获得当前head
            Node h = head;
            // head不为null并且head的等待状态不为0
            if (h != null && h.waitStatus != 0)
                // 唤醒下一个可以被唤醒的线程，不一定是next哦
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

- tryRelease(int)是AQS提供给子类实现的钩子方法，子类可以自定义实现独占式释放资源的方式，释放成功并返回true，否则返回false。
- unparkSuccessor(node)方法用于唤醒等待队列中下一个可以被唤醒的线程，不一定是下一个节点next，比如它可能是取消状态。
- head 的ws必须不等于0，为什么呢？当一个节点尝试挂起自己之前，都会将前置节点设置成SIGNAL -1，就算是第一个加入队列的节点，在获取锁失败后，也会将虚拟节点设置的 ws 设置成 SIGNAL，而这个判断也是防止多线程重复释放，接下来我们也能看到释放的时候，将ws设置为0的操作。

### void unparkSuccessor(Node node)

```java
    
	private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        // 如果node的waitStatus<0为signal，CAS修改为0
        // 将 head 节点的 ws 改成 0，清除信号。表示，他已经释放过了。不能重复释放。
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        // 唤醒后继节点，但是有可能后继节点取消了等待 即 waitStatus == 1
        Node s = node.next;
        // 如果后继节点为空或者它已经放弃锁了
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 从队尾往前找，找到没有没取消的所有节点排在最前面的【直到t为null或t==node才退出循环嘛】
            for (Node t = tail; t != null && t != node; t = t.prev)
                // 如果>0表示节点被取消了，就一直向前找呗，找到之后不会return，还会一直向前
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 如果后继节点存在且没有被取消，会走到这，直接唤醒后继节点即可
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

## 参考阅读

- [http://concurrent.redspider.group/article/02/11.html](http://concurrent.redspider.group/article/02/11.html)
- [一行一行源码分析清楚AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer)
- [【死磕 Java 并发】—– J.U.C 之 AQS：同步状态的获取与释放](http://cmsblogs.com/?p=2197)