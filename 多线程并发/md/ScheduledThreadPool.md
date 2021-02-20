[toc]

## ScheduledThreadPoolExecutor概述

我们在上一篇学习了ThreadPoolExecutor的实现原理：[Java并发包源码学习系列：线程池ThreadPoolExecutor源码解析](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/113786358)

本篇我们来学习一下在它基础之上的扩展：ScheduledThreadPoolExecutor。它继承了ThreadPoolExecutor并实现了ScheduledExecutorService接口，是一个可以在**指定一定延迟时间**后或者**定时进行任务调度**执行的线程池。

```java
public class TestScheduledThreadPool {

    private static final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    public static void main (String[] args) throws InterruptedException {
        scheduler.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run () {
                System.out.println("command .. " + new Date());
            }
        }, 0, 1, TimeUnit.SECONDS);
    }
}
```

简单看一个demo吧，这里使用Executors工具类创建ScheduledExecutorService，起始就是实例化了一个ScheduledThreadPoolExecutor，当然我们自定义也是可以的。

接着调用`scheduleAtFixedRate`方法，指定延迟为0，表示立即执行， 指定period为1，以1s为周期定时执行该任务。

**从整体感知ScheduledThreadPoolExecutor的执行**

1. 当调用scheduleAtFixedRate时，将会向延迟队列中添加一个任务ScheduledFutureTask。
2. 线程池中的线程从延迟队列中获取任务，并执行。

## 类图结构

![ScheduledThreadPoolExecutor](img/ScheduledThreadPool/ScheduledThreadPoolExecutor.png)

- 可以通过Executors工具类创建,也可以通过构造方法创建。

```java
    //Executors.java
	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
	//ScheduledThreadPoolExecutor.java
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

- ScheduledThreadPoolExecutor继承了ThreadPoolExecutor并实现了ScheduledExecutorService接口。
- 线程池队列使用DelayedWorkQueue，和DelayedQueue类似，是延迟队列。
- ScheduledFutureTask是一个具有返回值的任务，继承自FutureTask。

## ScheduledExecutorService

ScheduledExecutorService代表**可在指定延迟后或周期性地执行线程任务线程池**，提供了如下4个方法：

```java
public interface ScheduledExecutorService extends ExecutorService {
    
	// 指定command任务将在delay延迟后执行
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);
	// 指定callable任务将在delay延迟后执行
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);
	// 指定command任务将在delay延迟后执行，而且以设定频率重复执行
    // initialDelay + period 开始， initialDelay + n * period 处执行
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

	// 创建并执行一个在给定初始延迟后首次启用的定期操作，随后在每一次执行终止和下一次执行开始之间
    // 都存在给定的延迟。如果任务在任一一次执行时遇到异常，就会取消后续执行；
    // 否则，只能通过程序来显式取消或终止该任务
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);

}

```

## ScheduledFutureTask

可以按照DelayQueue中的Delayed的元素理解，是具体放入延迟队列中的东西，可以看到实现了getDelay和compareTo方法。

- getDelay获取元素剩余时间，也就是当前任务还剩多久过期，【剩余时间 = 到期时间 - 当前时间】。
- compareTo方法作为排序规则，一般规定最快过期的元素放到队首，q.peek()出来的就是最先过期的元素。

```java
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        /** FIFO队列中的序列号，time相同，序列号小的排在前面 */
        private final long sequenceNumber;

        /** 任务将要被执行的时间，也就是过期时间 */
        private long time;

        /**
         * period == 0  当前任务是一次性的， 执行完毕后就退出
         * period > 0   当前任务是fixed-delay任务，是固定延迟的定时可重复执行任务
         * period < 0   当前任务是fixed-rate任务，是固定频率的定时可重复执行任务
         */
        private final long period;

        /** The actual task to be re-enqueued by reExecutePeriodic */
        RunnableScheduledFuture<V> outerTask = this;

        /**
         * Index into delay queue, to support faster cancellation.
         */
        int heapIndex;
		
        //... 省略构造函数

        // 当前任务还剩多久过期
        public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), NANOSECONDS);
        }
		
        // 队列中的比较策略
        public int compareTo(Delayed other) {
            if (other == this) // compare zero if same object
                return 0;
            if (other instanceof ScheduledFutureTask) {
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time;
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                // time相同，序列号小的排在前面
                else if (sequenceNumber < x.sequenceNumber)
                    return -1;
                else
                    return 1;
            }
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }

		//... 省略其他方法
    }
```

## FutureTask

FutureTask内部使用一个state变量表示任务状态。

```java
public class FutureTask<V> implements RunnableFuture<V> {

    /**
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0; // 初始状态
    private static final int COMPLETING   = 1; // 执行中
    private static final int NORMAL       = 2; // 正常运行结束
    private static final int EXCEPTIONAL  = 3; // 运行中异常
    private static final int CANCELLED    = 4; // 任务被取消
    private static final int INTERRUPTING = 5; // 任务正在被中断
    private static final int INTERRUPTED  = 6; // 任务已经被中断
    
}
```

## schedule

提交一个延迟执行的任务，任务从提交时间算起延迟单位为unit的delay时间后开始执行。

如果提交的任务不是周期性的任务，任务只会执行一次。                     

```java
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        // 参数校验
        if (command == null || unit == null)
            throw new NullPointerException();
        // 任务转换： 把command任务转换为ScheduledFutureTask
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        // 添加任务到延迟队列
        delayedExecute(t);
        return t;
    }

	// 将延迟时间转换为绝对时间， 
    private long triggerTime(long delay, TimeUnit unit) {
        return triggerTime(unit.toNanos((delay < 0) ? 0 : delay));
    }
	// 将当前的那描述加上延迟的nanos后的long型值
    long triggerTime(long delay) {
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
    }
	
    private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
        
        ScheduledFutureTask(Runnable r, V result, long ns) {
            super(r, result); // 调用FutureTask的构造方法
            this.time = ns;
            this.period = 0; // 这里表示任务是一次性的
            this.sequenceNumber = sequencer.getAndIncrement();
        }
    }

// FutureTask.java
public class FutureTask<V> implements RunnableFuture<V> {
    public FutureTask(Runnable runnable, V result) {
        // 将runnable转化为callable
        this.callable = Executors.callable(runnable, result);
        // 设置当前的任务状态为NEW
        this.state = NEW;       // ensure visibility of callable
    }
}
```

### void delayedExecute(task)

1. 首先判断当前线程池是否已经关闭，如果已经关闭则执行线程池的拒绝策略，否则将任务添加到延迟队列。
2. 加入队列后，还要重新检查线程池是否被关闭，如果已经关闭则从延迟队列里删除刚才添加的任务，但此时可能线程池中的线程已经执行里面的任务，此时就需要取消该任务。

```java
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        // 如果线程池关闭， 则执行拒绝策略
        if (isShutdown())
            reject(task);
        else {
            // 将任务添加到延迟队列
            super.getQueue().add(task);
            // 检查线程池状态,如果已经关闭，则从延迟队列里面删除刚才添加的任务
            // 但此时可能线程池中的线程已经从任务队列里面移除了该任务
            // 此时需要调用cancel 取消任务
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                // 确保至少一个线程在处理任务
                ensurePrestart();
        }
    }
```

### boolean canRunInCurrentRunState(periodic)

判断当前任务是否应该被取消。

```java
    boolean canRunInCurrentRunState(boolean periodic) {
        return isRunningOrShutdown(periodic ?
                                   continueExistingPeriodicTasksAfterShutdown :
                                   executeExistingDelayedTasksAfterShutdown);
    }
```

periodic参数通过`isPeriodic()`得到，如果period为0，则为false。

相应的isRunningOrShutdown方法传入的参数就应该是executeExistingDelayedTasksAfterShutdown，默认为true，表示：其他线程调用了shutdown命令关闭线程池后，当前任务还是要执行

### void ensurePrestart()

 确保至少一个线程在处理任务：如果线程个数小于核心线程池数则新增一个线程，否则如果当前线程数为0，则新增一个线程。

```java
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        // 增加核心线程数
        if (wc < corePoolSize)
            addWorker(null, true);
        // 如果corePoolSize==0 也添加一个线程
        else if (wc == 0)
            addWorker(null, false);
    }
```

### ScheduledFutureTask#run()

具体执行任务的线程是Worker线程，任务执行是Worker线程调用任务的润方法执行，这里的任务是ScheduledFutureTask，也就是调用它的run方法。

```java
        public void run() {
            // 是否只执行一次 period != 0
            boolean periodic = isPeriodic();
            // 取消任务
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            // 任务只执行一次， 调用FutureTask的run
            else if (!periodic)
                ScheduledFutureTask.super.run();
            // 定时执行
            else if (ScheduledFutureTask.super.runAndReset()) {
                // 设置下一次运行时间
                setNextRunTime();
                // 重新加入延迟队列
                reExecutePeriodic(outerTask);
            }
        }
```

### FutureTask#run()

```java
    public void run() {
        // 如果任务不是NEW状态 直接返回
        // 如果是NEW， 但是cas设置当前任务的持有者为当前线程失败 也直接返回
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            // 再次判断任务的状态，避免两次判断状态之间有其他线程对任务状态进行修改
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 执行任务
                    result = c.call();
                    // 执行成功
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                // 如果执行任务成功
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

### FutureTask#set(V v)

```java
    protected void set(V v) {
        // CAS 将当前任务的状态 从 NEW 转化 为 COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            // 走到这里只有一个线程会到这里，设置任务状态 为NORMAL 正常结束
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```

### FutureTask#setException(Throwable t)

```java
    protected void setException(Throwable t) {
        // CAS 将当前任务的状态 从 NEW 转化 为 COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            // 走到这里只有一个线程会到这里，设置任务状态 为EXCEPTIONAL,非正常结束
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
```

## scheduleWithFixedDelay

针对任务类型为fixed-delay，当任务执行完毕后，让其延迟固定时间后再次运行，原理是：

1. 当向延迟队列中添加一个任务时，将会等待initialDelay时间，时间到了就过期，从队列中移除，并执行。
2. 执行完毕之后，会重新设置任务的延迟时间，然后再把任务放入延迟队列，循环。
3. 如果一个任务在执行过程中抛出了一个异常，任务结束，但不会影响其他任务的执行。

```java
    // initialDelay ： 提交任务后延迟多少时间开始执行任务
	// delay ： 当任务执行完毕后延长多少时间后再次运行任务
	public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        // 参数校验
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        // 任务转换 period < 0
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        // 添加任务到队列
        delayedExecute(t);
        return t;
    }
```

注意这里构造的ScheduledFutureTask的period<0，会导致`boolean periodic = isPeriodic();`的结果是true，因此在ScheduledFutureTask的run逻辑中，会调用FutureTask的runAndReset()方法。

### ScheduledFutureTask#run()

具体执行任务的线程是Worker线程，任务执行是Worker线程调用任务的润方法执行，这里的任务是ScheduledFutureTask，也就是调用它的run方法。

```java
        public void run() {
            // 是否只执行一次 period != 0
            boolean periodic = isPeriodic();
            // 取消任务
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            // 任务只执行一次， 调用FutureTask的run
            else if (!periodic)
                ScheduledFutureTask.super.run();
            // 定时执行
            else if (ScheduledFutureTask.super.runAndReset()) {
                // 设置下一次运行时间
                setNextRunTime();
                // 重新加入延迟队列
                reExecutePeriodic(outerTask);
            }
        }
```

### FutureTask#runAndReset()

相比于FutureTask的run方法，该方法逻辑差不多，但缺少了：在任务正常执行完后设置状态的步骤。原因在于：让任务成为可重复执行的任务。

```java
    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        // 如果当前任务正常执行完毕并且任务状态为NEW 则返回true， 否则返回false
        return ran && s == NEW;
    }
```

如果该方法返回true，将会调用setNextRunTime()设置下一次的运行时间，接着调用reExecutePeriodic(outerTask)重新加入任务队列。

### void setNextRunTime()

```java
		// 设置下一次运行时间
        private void setNextRunTime() {
            long p = period;
            if (p > 0)
                time += p;
            else
                // 延迟-p的时间
                time = triggerTime(-p);
        }
```

## scheduleAtFixedRate

针对任务类型为fixed-rate，相对起始时间点以固定频率调用指定的任务。

```java
    // initialDelay ： 提交任务后延迟多少时间开始执行任务
	// period 固定周期
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        // period > 0
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```

它和scheduleWithFixedDelay类似，区别在于：

1. period>0， 但仍然满足period!=0的条件。
2. setNextRunTime() 走进time+=p 的分支，而不是 time=triggerTime(-p)。

最终的执行规则为：**initialDelay + n * period** 的 刻执行任务，如果当前任务执行的时间到了，不会并发执行，下一次执行的任务将会延迟执行。

## 总结

- ScheduledThreadPoolExecutor内部使用DelayedWorkQueue存放执行的任务ScheduledFutureTask。
- ScheduledFutureTask是一个具有返回值的任务，继承自FutureTask。根据period的值分为三类：
  - period == 0 ，当前任务是**一次性**的，执行完毕后就退出。
  - period > 0 ，当前任务是fixed-delay任务，是**固定延迟**的定时可重复执行任务。
  - period < 0 ，当前任务是fixed-rate任务，是**固定频率**的定时可重复执行任务。

## 参考阅读

- 《Java并发编程之美》
- 《疯狂Java讲义》
- 《Java并发编程的艺术》