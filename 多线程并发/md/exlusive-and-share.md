## Java并发包源码学习系列：AQS共享模式获取与释放资源

往期回顾：

- [Java并发包源码学习系列：AbstractQueuedSynchronizer](https://www.cnblogs.com/summerday152/p/14238284.html)
- [Java并发包源码学习系列：CLH同步队列及同步资源获取与释放](https://www.cnblogs.com/summerday152/p/14244324.html)

上一篇文章介绍了AQS内置队列节点的出队入队操作，以及独占式获取共享资源与释放资源的详细流程，为了结构完整，本篇继续以AQS的角度介绍另外一种：共享模式获取与释放资源的细节，本篇暂不分析具体子类如ReentrantLock、ReentrantReadWriteLock的实现，之后会陆续补充。

## 独占式获取与释放资源的特点

当某个线程争夺同步资源失败之后，将会被包装为独占式节点，并加入CLH同步队列的队尾，并保持自旋。

同步队列中的线程在自旋时会判断其前驱节点是否为首节点，如果是首节点，它会尝试获取同步状态，获取成功则退出同步队列。

线程执行完逻辑之后，会释放同步状态，释放之后将会唤醒其后可被唤醒的后继节点。

> 友情提示：本篇文章着重介绍共享模式获取和释放资源的特点，许多代码实现上面和共享式和独占式其实逻辑差不多，为了清晰对比，这边会将独占式的部分核心代码粘贴过来，注意理解共享式和独占式存在差异的地方。详细解析可戳：[Java并发包源码学习系列：CLH同步队列及同步资源获取与释放](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/112301359)





## 共享式获取资源的方法

### void acquireShared(int arg)

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0) //子类实现
            doAcquireShared(arg);
    }
```

- `tryAcquireShared(int)`是AQS提供给子类实现的钩子方法，子类可以自定义实现**共享式获取资源**的方式，获取状态失败返回小于0。
- 如果获取失败，则进入`doAcquireShared(arg);`的逻辑。

### void doAcquireShared(int arg)

注意这里和独占式获取资源`acquireQueued`的区别。

```java
    private void doAcquireShared(int arg) {
        // 包装成共享模式的节点，入队
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            // 自旋
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    // 尝试获取同步状态，子类实现
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        // 设置新的首节点，并根据条件，唤醒下一个节点
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

我们可以看到有几个存在差异的地方：

1. 在共享式获取资源失败的时候，会包装成SHARED模式的节点入队。
2. 如果前驱节点为head，则使用tryAcquireShared方法尝试获取同步状态，这个方法由子类实现。
3. 如果获取成功r>=0，这时调用`setHeadAndPropagate(node, r)`，该方法首先会设置新的首节点，将第一个节点出队，接着会不断唤醒下一个共享模式节点，实现同步状态被多个线程共享获取。

接下来我们着重看下setHeadAndPropagate方法。

### void setHeadAndPropagate(Node node, int propagate)

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        // 节点出队，设置新的head
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        // propagate>0表示同步状态还可以被后面的节点获取
        // h.waitStatus<0表示该节点后面还有节点需要被唤醒
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

### doReleaseShared()

```java
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

