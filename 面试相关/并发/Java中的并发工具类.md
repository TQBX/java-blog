[toc]

# CountDownLatch

CountDownLatch允许一个或多个线程**等待其他线程完成操作**。类似于join的操作，可以进行类比：

> join用于让当前执行线程等待join线程执行结束，如`A.join()`方法，将不停检查A线程是否存活，如果A存活，则当前线程永远等待。

```java
public class JoinCDLT {

    public static void main(String[] args) throws InterruptedException {
        Thread parser1 = new Thread(new Runnable() {
            @SneakyThrows
            @Override
            public void run() {
                System.out.println("parser1 start");
                Thread.sleep(5000);
                System.out.println("parser1 finish");
            }
        });
        Thread parser2 = new Thread(new Runnable() {
            @SneakyThrows
            @Override
            public void run() {
                System.out.println("parser2 start");
                Thread.sleep(10000);
                System.out.println("parser2 finish");
            }
        });
        long start = System.currentTimeMillis();
        parser1.start();
        parser2.start();
        //join用于让当前执行线程等待join线程执行结束
        parser1.join();
        parser2.join();
        long end = System.currentTimeMillis();
        System.out.println("all parser finish , spend " + (end - start) + " ms");
    }
    
}
```

> CountDownLatch使用AQS实现，通过AQS的状态变量state来作为计数器值,当多个线程调用countdown方法时实际是原子性递减AQS的状态值，当线程调用await方法后当前线程会被放入AQS阻塞队列等待计数器为0再返回。

```java
public class CountDownLatchTest {

    //传入int参数作为计数器
    static CountDownLatch c = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        Thread a = new Thread(new Runnable() {
            @SneakyThrows
            @Override
            public void run() {
                Thread.sleep(3000);
                System.out.println(1);
                c.countDown();//每当调用该方法,计数器N - 1
            }
        });
        Thread b = new Thread(new Runnable() {
            @SneakyThrows
            @Override
            public void run() {
                Thread.sleep(3000);
                System.out.println(2);
                c.countDown();
            }
        });
        long start = System.currentTimeMillis();
        a.start();
        b.start();
        //await方法会阻塞当前线程
        c.await();
        long end = System.currentTimeMillis();
        System.out.println(3 + " " + (end - start));
    }
}
```

计数器必须大于等于0，只是等于0时候，计数器就是0，调用await方法时不会阻塞当前线程。CountDownLatch不可能重新初始化或者修改CountDownLatch对象的内部计数器的值。一个线程调用countDown方法happen-before，另外一个线程调用await方法。

# CyclicBarrier

CyclicBarrier可以**让一组线程达到一个屏障【同步点】时被阻塞**，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

> CyclicBarrier默认的构造方法是`CyclicBarrier(int parties)`，其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

```java
public class CyclicBarrierTest {

    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) {

        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //线程调用await,告诉CyclicBarrier已经到达了屏障,然后当前线程被阻塞
                    c.await(); 
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.println(1);
            }
        }, "thread-1");
        thread.start();
        try {
            //主线程到达了屏障,因为设置了parties设置为2,因此可以继续下去
            c.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
        System.out.println(2);
    }
}
```

上述程序，输出的结果可能是先1后2的次序，也可能是先2后1，原因在于：主线程和子线程的调度由CPU决定，两个线程都可能先执行。

但是，**如果将屏障数量改为3，此时主线程和子线程会永远等待，因为没有第三个线程达到屏障**了。



# CyclicBarrier和CountDownLatch的区别 

- CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置，因此CyclicBarrier能够处理更为复杂的业务场景。
- CyclicBarrier还提供了其他有用的方法，如`getNumberWaiting`方法可以获得CyclicBarrier阻塞的线程数量。`isBroken()`方法用来了解阻塞的线程是否被中断。

# Phaser 的实现

Phaser可以替代CountDownLatch 和CyclicBarrier，但比两者更加强大，可以动态调整需要的线程个数，可以通过构造函数传入父Phaser实现层次Phaser

# Semaphore 

Semaphore 可以用来**控制同时访问特定资源的线程数量**，它通过协调各个线程，以保证合理的使用公共资源，适用场景可以是流量控制。

使用AQS实现，AQS的状态变量state做为许可证数量，每次通过`acquire()/tryAcquire()`，许可证数量通过CAS原子性递减，调用release()释放许可证，原子性递增，只要有许可证就可以重复使用。

```java
public class SemaphoreTest {

    private static final int THREAD_COUNT = 30;
    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);

    private static Semaphore s = new Semaphore(10);//10个许可证数量,最大并发数为10

    public static void main(String[] args) {
        for(int i = 0; i < THREAD_COUNT; i ++){ //执行30个线程
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    s.tryAcquire(); //尝试获取一个许可证
                    System.out.println("save data");
                    s.release(); //使用完之后归还许可证
                }
            });
        }
        threadPool.shutdown();
    }
}
```

# Exchanger 原理

用于**进行线程间的数据交换**，它提供一个同步点，在这个同步点两个线程可以交换彼此的数据。

如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当都达到同步点时，这两个线程可以交换数据。