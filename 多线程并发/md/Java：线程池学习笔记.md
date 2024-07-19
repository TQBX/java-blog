[toc]

# 为什么要用线程池？

> 池化技术：减少每次获取资源的消耗，提高对资源的利用率。

**线程池**提供了一种限制和管理资源（包括执行一个任务）。 每个**线程池**还维护一些基本统计信息，例如已完成任务的数量。

使用线程池的好处：

- **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度**。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
- **提高线程的可管理性**。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

# 线程池的实现原理？

## execute方法源码

```java
    public void execute(Runnable command) {
        // 如果任务为null，则抛出异常。
        if (command == null)
            throw new NullPointerException();
        // ctl 中保存的线程池当前的一些状态信息  AtomicInteger
        int c = ctl.get();
        //判断当前线程池中执行的任务数量是否小于corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            //如果小于，则通过addWorker新建一个线程，然后，启动该线程从而执行任务。
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //通过 isRunning 方法判断线程池状态
        //线程池处于 RUNNING 状态才会被并且队列可以加入任务
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再次获取线程池状态，如果线程池状态不是 RUNNING 状态就需要从任务队列中移除任务。
            // 并尝试判断线程是否全部执行完毕。同时执行拒绝策略。
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果当前线程池为空就新创建一个线程并执行。
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //通过addWorker新建一个线程，并将任务(command)添加到该线程中；
        //然后，启动该线程从而执行任务。
        //如果addWorker执行失败，则通过reject()执行相应的拒绝策略的内容。
        else if (!addWorker(command, false))
            reject(command);
    }
```

> 线程池创建线程的时候，会将线程封装成工作线程Worker，Worker在执行完成任务之后，还会循环获取工作队列利的任务来执行。

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {//循环执行任务
                w.lock();
                //如果线程池正在停止，确保线程被中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

![ThreadPoolExecutor执行execute()方法](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/poll.png)

1. 线程池判断**核心线程池**【corePoolSize】里的线程是否都在执行任务。如果不是，则创建一个新的工作线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。
2. 线程池判断**工作队列**【BlockingQueue】是否已经满。如果工作队列没有满，则将新提交的任务存储在这个工作队列里。如果工作队列满了，则进入下个流程。 
3. 线程池判断**线程池**【maximumPoolSize】的线程是否都处于工作状态。如果没有，则创建一个新的工作线程来执行任务。如果已经满了，则交给**饱和策略**【RejectedExecutionHandler.rejectedExecution()】来处理这个任务。 

![ThreadPoolExecutor执行示意图 ](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/execute.png)

如何使用线程池？



# ThreadPoolExecutor重要分析

## 构造方法的重要参数

ThreadPoolExecutor方法的构造参数有很多，我们看看最长的那个就可以了：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

- `corePoolSize`：核心线程数定义了**最小可以同时运行的线程数量**。
- `maximumPoolSize`：**当队列中存放的任务达到队列容量的时候**，当前可以同时运行的线程数量变为**最大线程数**。【如果使用的无界队列，这个参数就没啥效果】
- `workQueue`: 当新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，**如果达到核心线程数的话，新任务就会被存放在队列中**。
- `keepAliveTime`:当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁。
- `unit`：`keepAliveTime` 的时间单位。
- `threadFactory`：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。
- `handler`：饱和策略，当前同时运行的线程数量达到最大线程数量【`maximumPoolSize`】并且队列也已经被放满时，执行饱和策略。

# 线程池的简单使用

```java
public class ThreadPoolTest {

    private static final int CORE_POOL_SIZE = 5; //核心线程数
    private static final int MAX_POOL_SIZE = 10; //最大线程数
    private static final int QUEUE_CAPACITY = 100; //任务队列的容量
    private static final Long KEEP_ALIVE_TIME = 1L; //等待时间

    public static void main(String[] args) {

        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(
                CORE_POOL_SIZE,
                MAX_POOL_SIZE,
                KEEP_ALIVE_TIME,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(QUEUE_CAPACITY),
                new ThreadPoolExecutor.AbortPolicy());
        for(int i = 0; i < 10 ; i ++){

            Runnable worker = new MyRunnable(""+ i); //创建任务
            threadPool.execute(worker); //通过execute提交
        }
        threadPool.shutdown();
        while(!threadPool.isTerminated()){

        }
        System.out.println("Finished all threads");


    }


}

class MyRunnable implements Runnable {
    private String command;

    MyRunnable(String s) {
        this.command = s;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " Start. Time = " + new Date());
        processCommand();
        System.out.println(Thread.currentThread().getName() + " End. Time = " + new Date());
    }

    private void processCommand() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public String toString() {
        return this.command;
    }
}
```

# 任务队列有哪些？

- ArrayBlockingQueue：基于数组结构的有界阻塞队列，按FIFO原则对元素进行排序。
- LinkedBlockingQueue：基于链表结构的阻塞队列，按FIFO排序元素，吞吐量通常要高于ArrayBlockingQueue，`Executors.newFixedThreadPool()`就是使用了这个队列。
- SynchronousQueue：一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，`Executors.newCachedThreadPool()`就是使用了这个队列。
- PriorityBlockingQueue：一个具有优先级的无限阻塞队列。

# 饱和策略有哪些呢？

- **`ThreadPoolExecutor.AbortPolicy`**：抛出 `RejectedExecutionException`来拒绝新任务的处理。【默认的饱和策略】

- **`ThreadPoolExecutor.CallerRunsPolicy`**【提供可伸缩队列】：调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，**如果执行程序已关闭，则会丢弃该任务**。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      if (!e.isShutdown()) {
          r.run();
      }
  }
  ```

- **`ThreadPoolExecutor.DiscardPolicy`：** 不处理新任务，直接丢弃掉。

- **`ThreadPoolExecutor.DiscardOldestPolicy`：** 此策略将丢弃最早的未处理的任务请求，并执行当前任务。

  ```java
  public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
      if (!e.isShutdown()) {
          e.getQueue().poll();
          e.execute(r);
      }
  }
  ```

当然，也可以根据需要自定义拒绝策略，需要实现`RejectedExecutionHandler`。

# Executor框架主要组成部分

![](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/Executor.png)

**任务**

- 包括被执行任务需要实现的接口：Runnable或Callable。

**任务的执行**

- 包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。
- Executor框架有两个关键类实现了ExecutorService接口 （ThreadPoolExecutor和ScheduledThreadPoolExecutor）。

**异步计算的结果**

- 包括接口Future和实现Future接口的FutureTask类。

# Executor如何使用

1. 主线程创建实现Runnable或者Callable接口的任务对象。【Executors可以实现Runnable对象和Callable对象的相互转化】

2. 将Runable对象交给ExecutorService执行。

   - `ExecutorService.execute(Runnable command)`

   - `ExecutorService.submit(Runnable task)`
   - `ExecutorService.submit(Callable<T> task)`

3. 如果执行了submit方法，将返回一个实现Future接口的对象。我们也可以创建FutureTask【实现了Future】，然后直接交给ExecutorService执行。
4. 最后主线程可以执行`FutureTask.get()`来等待任务执行完成，也可以执行`FutureTask.cancel(boolean mayInterruptIfRunning)`取消此任务的执行。

# 如何创建线程池？

一、使用ThreadPoolExecutor的各种构造方法。

二、通过Executor框架的工具类Executors可以创建三种类型的ThreadPoolExecutor。

《阿里巴巴 Java 开发手册》中**强制线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式**，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

> Executors 返回线程池对象的弊端如下：
>
> - **FixedThreadPool 和 SingleThreadExecutor** ： 允许请求的队列长度为 Integer.MAX_VALUE ，可能堆积大量的请求，从而导致 OOM。
> - **CachedThreadPool 和 ScheduledThreadPool** ： 允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致 OOM。



# 执行execute方法和submit方法的区别？

- **`execute()`方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功与否；**

```java
threadPool.execute(new Runnable() {
    @Override
    public void run() {

    }
}); //通过execute提交
```

- **`submit()`方法用于提交需要返回值的任务。线程池会返回一个 `Future` 类型的对象，通过这个 `Future` 对象可以判断任务是否执行成功**，并且可以通过 `Future` 的 `get()`方法来获取返回值，`get()`方法会阻塞当前线程直到任务完成，而使用 `get（long timeout，TimeUnit unit）`方法则会阻塞当前线程一段时间后立即返回，这时候有可能任务没有执行完。

```java
Future<Object> future = threadPool.submit(hasReturnValueTask);
try{
    Object s = future.get();
}catch(InterruptedException e){
    //处理中断异常
}catch(ExecutionException e){
    //处理无法执行任务异常
}finally{
    threadPool.shutdown();
}
```

# 几种Executors创建的常见线程池总结

## FixedThreadPool

可重用固定线程池数的线程池，任务队列使用的是无界的`LinkedBlockingQueue`。

FixedThreadPool运行示意图【图片来源《Java并发编程的艺术》】

![](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/fix.jpg)

1. 如果当前运行的线程数小于 corePoolSize， 如果再来新任务的话，就创建新的线程来执行任务；
2. 当前运行的线程数等于 corePoolSize 后， 如果再来新任务的话，会将任务加入 `LinkedBlockingQueue`；
3. 线程池中的线程执行完 手头的任务后，会在循环中反复从 `LinkedBlockingQueue` 中获取任务来执行；

**不推荐使用FixedThreadPool的原因**

## SingleThreadExecutor

**SingleThreadExecutor是只有一个线程的线程池。**

SingleThreadExecutor运行示意图【图片来源《Java并发编程的艺术》】

![](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/single.jpg)

1. 如果当前运行的线程数少于 corePoolSize，则创建一个新的线程执行任务；
2. 当前线程池中有一个运行的线程后，将任务加入 `LinkedBlockingQueue`；
3. 线程执行完当前的任务后，会在循环中反复从`LinkedBlockingQueue` 中获取任务来执行；

**不推荐使用SingleThreadPool的原因**

同FixedThreadPool，任务很多时，可能会引发OOM。

## CacheThreadPool

`CachedThreadPool` 是一个会根据需要创建新线程的线程池，使用的任务队列是：`SynchronousQueue`。

![](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/cache.jpg)

1. 首先执行 `SynchronousQueue.offer(Runnable task)` 提交任务到任务队列。如果当前 `maximumPool` 中有线程正在执行 `SynchronousQueue.poll(keepAliveTime,TimeUnit.NANOSECONDS)`，那么主线程执行 offer 操作与空闲线程执行的 `poll` 操作配对成功，主线程把任务交给空闲线程执行，`execute()`方法执行完成，否则执行下面的步骤 2；
2. 当初始 `maximumPool` 为空，或者 `maximumPool` 中没有空闲线程时，将没有线程执行 `SynchronousQueue.poll(keepAliveTime,TimeUnit.NANOSECONDS)`。这种情况下，步骤 1 将失败，此时 `CachedThreadPool` 会创建新线程执行任务，execute 方法执行完成；

**不推荐使用CachedThreadPool的原因**

`CachedThreadPool`允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致 OOM。

## ScheduledThreadPoolExecutor

使用的是DelayedWorkQueue，ScheduledThreadPoolExecutor会把待调度的任务 （ScheduledFutureTask）放到一个DelayQueue中。

DelayQueue封装了一个PriorityQueue，这个PriorityQueue会对队列中的Scheduled-FutureTask进行排序。排序时，time小的排在前面（时间早的任务将被先执行）。如果两个ScheduledFutureTask的time相同，就比较sequenceNumber，sequenceNumber小的排在前面（也就是说，如果两个任务的执行时间相同，那么先提交的任务将被先执行）。

![](img/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E7%9F%A5%E8%AF%86%E7%82%B9/sech.png)

1. 线程1从DelayQueue中获取已到期的ScheduledFutureTask（DelayQueue.take()）。 到期任务是指ScheduledFutureTask的time大于等于当前时间。
2. 线程1执行这个ScheduledFutureTask。
3. 线程1修改ScheduledFutureTask的time变量为下次将要被执行的时间。
4. 线程1把这个修改time之后的ScheduledFutureTask放回DelayQueue中（DelayQueue.add()）。

## WorkStealingPool

创建一个含有足够多线程的线程池，能够调用闲置的CPU去处理其他的任务，使用ForkJoinPool实现，jdk8新增。

# LinkedBlockingQueue与ArrayBlockingQueue

> 线程池的阻塞队列为什么都用LinkedBlockingQueue，而不用ArrayBlockingQueue

LinkedBlockingQueue 使用单向链表实现，在声明的时候，**可以不指定队列长度**，长度为Integer.MAX_VALUE, 并且新建了一个Node对象,Node对象具有item，next变量，item用于存储元素，next指向链表下一个Node对象，在刚开始的时候链表的head,last都指向该Node对象，item、next都为null,新元素放在链表的尾部，并从头部取元素。取元素的时候只是一些指针的变化，**LinkedBlockingQueue给put(放入元素)，take(取元素)都声明了一把锁，放入和取互不影响，效率更高**。

ArrayBlockingQueue 使用数组实现，在声明的时候**必须指定长度**，如果长度太大，造成内存浪费，长度太小，并发性能不高，如果数组满了，就无法放入元素，除非有其他线程取出元素，**放入和取出都使用同一把锁**，因此存在竞争，效率比LinkedBlockingQueue低

# 线程池不使用的时候，需要关闭吗？

- 线程池的作用确实是为了减少频繁创建线程，以达到线程复用的目的。
- 但是如果不使用线程池的时候，线程池中的核心线程依然会一直存在，导致资源浪费，因此，在不使用线程池的时候可以通过shutdown方法关闭线程池。

# 如何合理配置Java线程池

> 线程数量太小，可能导致大量任务在队列中排队等待执行，最终OOM，CPU无法得到充分利用。
>
> 线程数量太大，大量线程可能会同时争取CPU的资源，导致大量的上下文切换。

**计算公式**

- **CPU 密集型任务(N+1)：** 这种任务消耗的主要是 CPU 资源，可以**将线程数设置为 N（CPU 核心数）+1**，比 CPU 核心数多出来的一个线程是为了**防止线程偶发的缺页中断**，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
- **I/O 密集型任务(2N)：** 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，**而线程在处理 I/O 的时间段内不会占用 CPU 来处理**，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

**如何判断是 CPU 密集任务还是 IO 密集任务？**

CPU 密集型简单理解就是利用 CPU 计算能力的任务比如你在内存中对大量数据进行排序。但凡涉及到网络读取，文件读取这类都是 IO 密集型，这类任务的特点是 CPU 计算耗费时间相比于等待 IO 操作完成的时间来说很少，大部分时间都花在了等待 IO 操作完成上。





参考：

- [https://snailclimb.gitee.io/javaguide/#/./docs/java/Multithread/best-practice-of-threadpool](https://snailclimb.gitee.io/javaguide/#/./docs/java/Multithread/best-practice-of-threadpool)
- 《Java并发编程的艺术》