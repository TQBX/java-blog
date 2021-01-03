JDK1.5出现应对高并发的包。

BlockingQueue ConcurrentMap ExecutorService Lock Atomic

## BlockingQueue阻塞队列

### ArrayBlockingQueue

基于数组，需要指定容量。

```java
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);
        // 队列满抛出异常
        queue.add(1); 
        // 队列满不会抛异常,返回false
        queue.offer(1);
        //队列满不抛错,而是阻塞,直到有元素被取出
        queue.put(1);
        // 队列如果满,阻塞5s
        queue.offer(1,5, TimeUnit.SECONDS);

        // 队列为空,抛出异常
        queue.remove();
        // 队列为空,返回null
        queue.poll();
        // 队列为空,阻塞
        queue.take();
        // 队列为空,定时阻塞
        queue.poll(5,TimeUnit.SECONDS);
    }
```

### LinkedBlockingQueue

底层基于链表。

可以指定容量，也可以不指定，如果不指定，为Integer.MAX_VALUE，一般认为是无界的。

```java
    public LinkedBlockingDeque() {
        this(Integer.MAX_VALUE);
    }
```

### PriorityBlockingQueue

具有优先级的阻塞队列

不指定容量，默认为11。

遍历队列式，对放入其中的元素进行自然排序，迭代排序则不会保证顺序。

要求置入的元素必须实现Comparable。

### SynchronousQueue

同步队列

不需要指定容量，容量默认为1且只能为1。

## ConcurrentMap



## ExecutorService



## Lock



