## 什么是阻塞队列

> 阻塞队列 = 阻塞 + 队列。

- 队列：一种**先进先出**的数据结构，支持尾部添加、首部移除或查看等基础操作。

- 阻塞：除了队列提供的基本操作之外，还提供了支持**阻塞式插入和移除**的方式。

下面这些对BlockingQueue的介绍基本翻译自JavaDoc，非常详细。

1. 阻塞队列的顶级接口是`java.util.concurrent.BlockingQueue`,它继承了Queue，Queue又继承自Collection接口。
2. BlockingQueue 对插入操作、移除操作、获取元素操作提供了四种不同的方法用于不同的场景中使用：1、抛出异常；2、返回特殊值（null 或 true/false，取决于具体的操作）；3、阻塞等待此操作，直到这个操作成功；4、阻塞等待此操作，直到成功或者超时指定时间，第二节会有详细介绍。
3. BlockingQueue不接受null的插入，否则将抛出空指针异常，因为poll失败了会返回null，如果允许插入null值，就无法判断poll是否成功了。
4. BlockingQueue可能是有界的，如果在插入的时候发现队列满了，将会阻塞，而无界队列则有`Integer.MAX_VALUE`大的容量，并不是真的无界。
5. BlockingQueue通常用来作为生产者-消费者的队列的，但是它也支持Collection接口提供的方法，比如使用remove(x)来删除一个元素，但是这类操作并不是很高效，因此尽量在少数情况下使用，如：当一条入队的消息需要被取消的时候。
6. BlockingQueue的实现都是线程安全的，所有队列的操作或使用内置锁或是其他形式的并发控制来保证原子。但是一些批量操作如：`addAll`,`containsAll`, `retainAll`和`removeAll`不一定是原子的。如 addAll(c) 有可能在添加了一些元素后中途抛出异常，此时 BlockingQueue 中已经添加了部分元素。
7. BlockingQueue不支持类似close或shutdown等关闭操作。

```java
// Doug Lea： BlockingQueue 可以用来保证多生产者和消费者时的线程安全
class Producer implements Runnable{
    private final BlockingQueue queue;
    Producer(BlockingQueue q){ queue = q; }
    public void run(){
        try{
            while(true) { queue.put(produce()); }
        }catch(InterruptedException ex){ ...handle... }
    }
    Object produce() { ... }
}

class Consumer implements Runnable{
    private final BlockingQueue queue;
    Consumer(BlockingQueue q){ queue = q; }
    public void run(){
        try{
            while(true) { consume(queue.take())); }
        }catch(InterruptedException ex){ ...handle... }
    }
    void consume(Object x) { ... }
}

class Setup{
    void main(){
        BlockingQueue q = new SomeQueueImplementation();
        Producer p = new Producer(q);
        Consumer c1 = new Consumer(q);
        Consumer c2 = new Consumer(q);
        new Thread(p).start();
        new Thread(c1).start();
        new Thread(c2).start();
    }
}
```

## 阻塞队列提供的方法

BlockingQueue 对插入操作、移除操作、获取元素操作提供了四种不同的方法用于不同的场景中使用：

| 方法类别 | 抛出异常  | 返回特殊值 | 一直阻塞 | 超时退出               |
| -------- | --------- | ---------- | -------- | ---------------------- |
| 插入     | add(e)    | offer(e)   | `put(e)` | `offer(e, time, unit)` |
| 移除     | remove()  | poll()     | `take()` | `poll(time, unit)`     |
| 瞅一瞅   | element() | peek()     |          |                        |

博主在这边大概解释一下，如果队列可用时，上面的几种方法其实效果都差不多，但是当队列空或满时，会表现出部分差异：

1. 抛出异常：当队列满时，如果再往队列里add插入元素e时，会抛出`IllegalStateException: Queue full`的异常，如果队空时，往队列中取出元素【移除或瞅一瞅】会抛出`NoSuchElementException`异常。
2. 返回特殊值：队列满时，offer插入失败返回false。队列空时，poll取出元素失败返回null，而不是抛出异常。
3. 一直阻塞：当队列满时，put试图插入元素，将会一直阻塞插入的生产者线程，同理，队列为空时，如果消费者线程从队列里take获取元素，也会阻塞，知道队列不为空。
4. 超时退出：可以理解为一直阻塞情况的超时版本，线程阻塞一段时间，会自动退出阻塞。

我们本篇的重点是阻塞队列，那么【一直阻塞】和【超时退出】相关的方法是我们分析的重头啦。

## 参考阅读

- 方腾飞《并发编程的艺术》