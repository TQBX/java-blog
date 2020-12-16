# 使用iterator遍历集合时的异常集合

## ConcurrentModificationException

网上关于集合类型使用Iterator遍历需要注意的事项想必大家都已熟知，如果你想要遍历的时候删除集合中的元素，如果你像下面这样写，是会报错的！

```java
    public void testRemove() {
        Iterator<String> iterator = list.iterator();
        while(iterator.hasNext()){
            String next = iterator.next();
            if("1".equals(next)){
                list.remove(next);//引发ConcurrentModificationException
            }
        }
    }
```

异常如下：

```java
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
	at java.util.ArrayList$Itr.next(ArrayList.java:859)
	at list.ListTest.testRemove(ListTest.java:40)
	at list.ListTest.main(ListTest.java:33)
```

Iterator迭代器采用`fail-fast`机制，一旦在迭代过程中检测到该集合已经被修改，程序会立即引发：`ConcurrentModificationException`异常，以避免共享资源而引发的潜在问题。

正确的写法是使用iterator提供的remove方法：

```java
    public void testCorrectRemove(){
        Iterator<String> iterator = list.iterator();
        while(iterator.hasNext()){
            String next = iterator.next();
            if("1".equals(next)){
                iterator.remove(); // 使用迭代器的remove
            }
        }
        System.out.println(list); 
    }
```

## UnsupportedOperationException

但是今天在测试的时候遇到一个问题，我的代码如下：

```java
    public static void main(String[] args) {
		// 注意这里创建list的方式
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
        Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()) {
            Integer num = iterator.next();
            System.out.println(num);
            if (num == 2) {
                iterator.remove();
            }
        }
        System.out.println(list);
    }
```

引起了如下异常：

```java
Exception in thread "main" java.lang.UnsupportedOperationException
	at java.util.AbstractList.remove(AbstractList.java:161)
	at java.util.AbstractList$Itr.remove(AbstractList.java:374)
	at list.ListTest.main(ListTest.java:28)
```

这就奇怪了，为什么这个List不行呢？

```java
    public static <T> List<T> asList(T... a) {
        return new ArrayList<>(a);
    }
```

明明创建的就是一个ArrayList啊，事实上，此ArrayList非彼ArrayList，我们可以跟进去谈谈究竟：

```java
    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable
    {
        private static final long serialVersionUID = -2764017481108945198L;
        private final E[] a;

        ArrayList(E[] array) {
            a = Objects.requireNonNull(array);
        }
    }
```

这个ArrayList是定义在Arrays类中的一个静态内部类，和我平时使用的并不是同一个！并且，我们可以看看其中Iterator的remove方法：

```java
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                // 调用本来的remove方法
                AbstractList.this.remove(lastRet);
                if (lastRet < cursor)
                    cursor--;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException e) {
                throw new ConcurrentModificationException();
            }
        }
```

！它调用了本类的remove方法，而remove方法并没有具体实现，而是抛出了异常：

```java
    public E remove(int index) {
        throw new UnsupportedOperationException();
    }
```

而我们平时的ArrayList的remove方法的实现是下面这个样子的：

```java
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

这下子，恍然大悟，原来是这样，了解这个之后，修改起来就相对简单了，我们把它转化为我们熟知的ArrayList就好了：

```java
    public static void main(String[] args) {
		// 转化为ArrayList
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Iterator<Integer> iterator = list.iterator();
        while (iterator.hasNext()) {
            Integer num = iterator.next();
            System.out.println(num);
            if (num == 2) {
                iterator.remove();
            }
        }
        System.out.println(list);
    }
```

## 移除指定数值

先看个例子：

```java
    public void testRemoveNumber() {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        list.remove(3);
        System.out.println(list);
    }
```

如果程序执行，结果会是如何呢？答案如下：

```java
[1, 2, 3, 5]  // 移除了index = 3的元素，而不是移除值为3的元素
```

如果我就是想移除值为3的元素呢？可以使用Integer包装一下。

```java
	public void testRemoveNumber() {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        list.remove(new Integer(3));
        System.out.println(list);
    }
```

结果就变成了：

```java
[1, 2, 4, 5] // 移除了值为3元素
```

两者区别在于：**调用的remove方法不同**。前者调用的是：`remove(int index)`，后者调用的是`remove(Object o)`。因此，如果我们想要移除某个值，且这个值是数值的时候，我们需要注意一下这个问题。

