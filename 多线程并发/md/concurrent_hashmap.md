注：本篇基于JDK1.8。

## 为什么要使用ConcurrentHashMap?

在思考这个问题之前，我们可以思考：如果不用ConcurrentHashMap的话，有哪些其他的容器供我们选择呢？并且它们的缺陷是什么？

哈希表利用哈希算法能够花费O(1)的时间复杂度高效地根据key找到value值，能够满足这个需求的容器还有HashTable和HashMap。

**HashTable**

HashTable使用synchronized关键字保证了多线程环境下的安全性，但加锁的实现方式是独占式的，所有访问HashTable的线程都必须竞争同一把锁，性能较为低下。

```java
    public synchronized V put(K key, V value) {
        // ...
    }
```

**HashMap**

JDK1.8版本的HashMap在读取hash槽的时候读取的是工作内存中引用指向的对象，在多线程环境下，其他线程修改的值不能被及时读到。

- 关于HashMap的源码解析：[【JDK1.8】Java集合源码学习系列：HashMap源码详解](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/104095675)

> 这就引发出可能存在的一些问题：
>
> 比如在插入操作的时候，**第一次将会根据key的hash值判断当前的槽内是否被占用，如果没有的话就会插入value**。在并发环境下，如果A线程判断槽未被占用，在执行写入操作的时候正巧时间片耗尽，此时B线程正巧也执行了同样的操作，率先插入了B的value，此时A正巧被CPU重新调度继续执行写入操作，进而将线程B的value覆盖。
>
> 还有一种情况是在同一个hash槽内，HashMap总是保持key唯一，在插入的时候，如果存在key，就会进行value覆盖。并发情况下，如果A线程判断最后一个节点仍未发现重复的key，那么将会执行插入操作，如果B线程在A判断和插入之间执行了同样的操作，也会发生数据的覆盖，也就是数据的丢失。
>
> 当然，像这样的并发问题其实还有一些，这里就不细说了，刚兴趣的小伙伴可以查阅下资料。

## ConcurrentHashMap的结构特点

### Java8之前

在Java8之前，底层采用Segment+HashEntry的方式实现。

采用分段锁的概念，底层使用Segment数组，Segment通过继承ReentrantLock来进行加锁，每次需要加锁的操作会锁住一个segment，分段保证每个段是线程安全的。

> 图源：[Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析](https://javadoop.com/post/hashmap)

![image-20210122212557376](img/concurrent_hashmap/image-20210122212557376.png)

### Java8之后

JDK1.8之后采用`CAS + Synchronized`的方式来保证并发安全。

采用【Node数组】加【链表】加【红黑树】的结构，与HashMap类似。

> 图源：[Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析](https://javadoop.com/post/hashmap)

![image-20210122212447837](img/concurrent_hashmap/image-20210122212447837.png)

## 基本常量

过一遍即可，不用过于纠结，有些字段也许是为了兼容Java8之前的版本。

```java
    /* ---------------- Constants -------------- */

    //  允许的最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认初始值16，必须是2的幂
    private static final int DEFAULT_CAPACITY = 16;

    // toArray相关方法可能需要的量
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //为了和Java8之前的分段相关内容兼容，并未使用
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    // 负载因子
    private static final float LOAD_FACTOR = 0.75f;

    // 链表转红黑树阀值> 8 链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;

    // 树转链表阀值，小于等于6（tranfer时，lc、hc=0两个计数器分别++记录原bin、新binTreeNode数量，<=UNTREEIFY_THRESHOLD 则untreeify(lo)）
    static final int UNTREEIFY_THRESHOLD = 6;

    // 链表树化的最小容量 treeifyBin的时候，容量如果不足64，会优先选择扩容到64
    static final int MIN_TREEIFY_CAPACITY = 64;

    // 每一步最小重绑定数量
    private static final int MIN_TRANSFER_STRIDE = 16;

    // sizeCtl中用于生成标记的位数
    private static int RESIZE_STAMP_BITS = 16;

    // 2^15-1，help resize的最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    // 32-16=16，sizeCtl中记录size大小的偏移量
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    // forwarding nodes的hash值
    static final int MOVED     = -1;

    // 树根节点的hash值
    static final int TREEBIN   = -2;

    // ReservationNode的hash值
    static final int RESERVED  = -3;

    // 提供给普通node节点hash用
    static final int HASH_BITS = 0x7fffffff;

    // 可用处理器数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();
```

## 重要成员变量

```java
    /* ---------------- Fields -------------- */

	// 就是我们说的底层的Node数组，懒初始化的，在第一次插入的时候才初始化，大小需要是2的幂
    transient volatile Node<K,V>[] table;

    /**
     * 扩容resize的时候用的table
     */
    private transient volatile Node<K,V>[] nextTable;

    /**
     * 基础计数器，是通过CAS来更新的
     */
    private transient volatile long baseCount;

    /**
     * Table initialization and resizing control.  
     * 如果为负数：表示正在初始化或者扩容，具体如下、
     * -1表示初始化，
     * -N，N-1表示正在进行扩容的线程数
     
     * 默认为0，初始化之后，保存下一次扩容的大小
     */
    private transient volatile int sizeCtl;

    /**
     * 扩容时分割table的索引
     */
    private transient volatile int transferIndex;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
    private transient volatile int cellsBusy;

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells;

    // 视图
    private transient KeySetView<K,V> keySet;
    private transient ValuesView<K,V> values;
    private transient EntrySetView<K,V> entrySet;
```

## 构造方法

和HashMap一样，table数组的初始化是在第一次插入的时候才进行的。

```java
    /**
     * 创建一个新的，空的map，默认大小为16
     */
    public ConcurrentHashMap() {
    }

    /**
     * 指定初始容量
     */
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
					// hashmap中讲过哦，用来返回的是大于等于传入值的最小2的幂次方
                   // https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/104095675#tableSizeFor_105
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        // sizeCtl 初始化为容量
        this.sizeCtl = cap;
    }

    /**
     * 接收一个map对象
     */
    public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }

    /**
     * 指定初始容量和负载因子
     * @since 1.6
     */
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    /**
     * 最全的：容量、负载因子、并发级别
     */
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

## put方法存值

### putVal

```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 对key进行散列计算 ： (h ^ (h >>> 16)) & HASH_BITS;
        int hash = spread(key.hashCode()); 
        // 记录链表长度
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            //第一次put，就是这里进行初始化的
            if (tab == null || (n = tab.length) == 0) 
                tab = initTable();
            // 找该 hash 值对应的数组下标，得到第一个节点 f， 
            // 这里判断f是否为空，就是这个位置上有没有节点占着
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果没有，用CAS尝试将值放入，插入成功，则退出for循环
                // 如果CAS失败，则表示存在并发竞争，再次进入循环
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // hash是Node节点f的一个属性，后面再分析
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                // 进入这个分支表示：根据key计算出的hash，得到的位置上是存在Node的，接着我将遍历链表了
                V oldVal = null;
                // 锁住Node节点
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) { // 头节点的hash属性 >= 0
                            binCount = 1; // 记录链表长度
                            // 链表的遍历操作，你懂的
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 找到一样的key了
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    // onlyIfAbsent为false的话，这里就要覆盖了，默认是覆盖的
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    // 接着也就结束遍历了
                                    break;
                                }
                                Node<K,V> pred = e;
                                // 遍历到最后了，把Node插入尾部
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) { // 红黑树
                            Node<K,V> p;
                            binCount = 2;
                            // 红黑树putTreeVal
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    // 判断是否需要将链表转换为红黑树 节点数 >= 8的时候
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

其实你会发现，如果你看过HashMap的源码，理解ConcurrentHashMap的操作其实还是比较清晰的，相信看下来你已经基本了解了。接下来将会具体分析一下几个关键的方法

### initTable

采用延迟初始化，第一次put的时候，调用initTable()初始化Node数组。

```java
    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 如果小于0，表示已经有其他线程抢着初始化了
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 这里就试着cas抢一下，抢到就将sc设置为-1，声明主权，抢不到就再次进入循环
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // 设置初始容量
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        // 创建容量为n的数组
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        // 赋值给volatile变量table底层数组
                        table = tab = nt;
                        // 这里其实就是 sc = n - n/4 = 0.75 * n
                        sc = n - (n >>> 2);
                    }
                } finally {
                    // 就当12吧
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

初始化的并发问题如何解决呢？

通过volatile的sizeCtl变量进行标识，在第一次初始化的时候，如果有多个线程同时进行初始化操作，他们将会判断sizeCtl是否小于0，小于0表示已经有其他线程在进行初始化了。

因为获取到初始化权的线程已经通过cas操作将sizeCtl的值改为-1了，且volatile的特性保证了变量在各个线程之间的可见性。

接着，将会创建合适容量的数组，并将sizeCtl的值设置为`cap*loadFactor`。

## get方法取值

### get

get方法相对来说就简单很多了，根据key计算出的hash值，找到对应的位置，判断头节点是不是要的值，不是的话就从红黑树或者链表里找。

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // 计算hash值
        int h = spread(key.hashCode());
        // 找到对应的hash桶的位置
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 正好在头节点上
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 树形结构上
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 在链表上
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

## 参考阅读

- [占小狼： 谈谈ConcurrentHashMap1.7和1.8的不同实现](https://www.jianshu.com/p/e694f1e868ec)

- [jianshu_xw：浅析JDK1.8 下HashMap线程安全性](https://www.jianshu.com/p/06e7e626dcad)