Java8时Lambda表达式的出现，将行为作为参数传递进函数的函数式编程，大大简化了之前冗杂的写法。

如果你对Lambda还不了解，可以参考我之前的关于Lambda表达式的总结：[                             Java8的Lambda表达式，你会不？                       ](https://www.cnblogs.com/summerday152/p/14095103.html)

对于集合一类，我们来整理一下发生的变化叭。

## Iterable的forEach

Iterable接口就是所有可迭代类型的父接口，我们熟知的Collection接口就是继承自它。Java8接口默认方法以及Lambda表达式的出现，让我们在遍历元素时对元素进行操作变得格外简单。

下面的forEach方法就是Java8新增的，它接受一个Consumer对象，是一个消费者类型的函数式接口。

```java
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
```

下面这段代码遍历输出每个元素。

```java
    public void testForEach(){
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        list.forEach(System.out::println);
    }
```

## Iterator的forEachRemaining

`java.util.Iterator`是用于遍历集合的迭代器，接口定义如下：

```java
public interface Iterator<E> {

    boolean hasNext();

    E next();

    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

default方法是Java8接口中新增的，`forEachRemaining`方法接收一个Consumer，我们可以通过该方法简化我们的遍历操作：

```java
    /**
     * Java8 为Iterator新增了 forEachRemaining(Consumer action) 方法
     */
    public static void main(String[] args) {

        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Iterator<Integer> iterator = list.iterator();
        iterator.forEachRemaining(System.out::println);
    }
```

## Collection的removeIf

Java8为Collection增加了默认的removeIf方法，接收一个Predicate判断，test方法结果为true，就移除对应的元素。

```java
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            // 判断元素是否需要被移除
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
```

下面这段代码移除列表中的偶数元素。

```java
    public void testRemoveIf() {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        list.removeIf(x -> x % 2 == 0);
        list.forEach(System.out::println);
    }
```

## Stream操作

具体使用可以参照[Java8的StreamAPI常用方法总结](https://www.cnblogs.com/summerday152/p/14118153.html)

```java
    public void testStream(){
        IntStream stream = IntStream.builder().add(1).add(2).add(3).build();
        int max = stream.max().getAsInt();
        System.out.println(max);
    }
```

## List的replaceAll

Java8为List接口增加了默认的replaceAll方法，需要UnaryOperator来替换所有集合元素，UnaryOperator是一个函数式接口。

```java
    default void replaceAll(UnaryOperator<E> operator) {
        Objects.requireNonNull(operator);
        final ListIterator<E> li = this.listIterator();
        while (li.hasNext()) {
            li.set(operator.apply(li.next()));
        }
    }
```

下面这个示例为每个元素加上3。

```java
    public void testReplaceAll(){
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        list.replaceAll(x -> x + 3);
        list.forEach(System.out::println);
    }
```

## List的sort

Java8为List接口增加了默认的sort方法，需要Comparator对象来控制元素排，我们可以传入Lambda表达式。

```java
    default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {
            i.next();
            i.set((E) e);
        }
    }
```

下面这个例子将list逆序。

```java
    public void testSort() {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        list.sort((x, y) -> y - x);
        System.out.println(list);
    }
```

## Map的ForEach

Map接口在Java8同样也新增了用于遍历的方法：

```java
    default void forEach(BiConsumer<? super K, ? super V> action) {
        Objects.requireNonNull(action);
        for (Map.Entry<K, V> entry : entrySet()) {
            K k;
            V v;
            try {
                k = entry.getKey();
                v = entry.getValue();
            } catch(IllegalStateException ise) {
                // this usually means the entry is no longer in the map.
                throw new ConcurrentModificationException(ise);
            }
            action.accept(k, v);
        }
    }
```

该方法接收一个二元的参数，分别对应key和value：

```java
    private void print(Map<Integer,String> map){
        map.forEach((x , y )-> {
            System.out.println("x -> " + x + ", y -> " + y);
        });
    }
```

为了接下来测试方便，数据先准备一下：

```java
    Map<Integer, String> map = new HashMap<>();

    {
        map.put(1, "hello");
        map.put(2, "summer");
        map.put(3, "day");
        map.put(4, "tqbx");
    }
```

## Map的remove

Java8新增了一个`remove(Object key, Object value)`方法。

```java
    default boolean remove(Object key, Object value) {
        // key存在且key对应的value确实是传入的value才移除
        Object curValue = get(key);
        if (!Objects.equals(curValue, value) ||
            (curValue == null && !containsKey(key))) {
            return false;
        }
        remove(key);
        return true;
    }
```

移除key为3且value为day的元素。

```java
    @Test
    public void testRemove(){
        map.remove(3,"day");
        print(map);
    }
```

## Map的compute相关方法

`Object compute(Object key, BiFunction remappingFunction)`：该方法使用remappingFunction根据key-value对计算一个新value，情况如下：

- 只要新value不为null，就使用新value覆盖原value。
- 如果原value不为null，但新value为null，则删除原key-value对。
- 如果原value、新value同时为null那么该方法不改变任何key-value对，直接返回null。

`Object computeIfAbsent(Object key, Function mappingFunction)`：

- 如果传给该方法的key参数在Map中对应的value为null，则使用mappingFunction根据key计算个新的结果。
- 如果计算结果不为null，则用计算结果覆盖原有的value。
- 如果原Map原来不包括该key，那么该方法可能会添加一组key-value对。

`Object computeIfPresent(Object key, BiFunction remappingFunction)`：

- 如果传给该方法的key参数Map中对应的value不为null，该方法将使用remappingFunction根据原key、value计算一个新的结果。
- 如果计算结果不为null，则使用该结果覆盖原来的value。
- 如果计算结果为null，则删除原key-value对。

```java
    @Test
    public void testCompute() {
        //key==2 对应的value存在时，使用计算的结果作为新value
        map.computeIfPresent(2, (k, v) -> v.toUpperCase());
        print(map);

        //key==6 对应的value为null (或不存在)时，使用计算的结果作为新value
        map.computeIfAbsent(6, (k) -> k + "haha");
        print(map);
    }
```

## Map的getOrDefault

```java
    default V getOrDefault(Object key, V defaultValue) {
        V v;
        // key存在或 value存在，则返回对应的value，否则返回defaultValue
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
```

```java
    @Test
    public void testGetOrDefault(){
        // 获取指定key的value，如果该key不存在，则返回default
        String value = map.getOrDefault(5, "如果没有key==5的value,就返回这条信息");
        System.out.println(value);
    }
```

## Map的merge

```java
    default V merge(K key, V value,
            BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        Objects.requireNonNull(remappingFunction);
        Objects.requireNonNull(value);
        // 先获取原值
        V oldValue = get(key);
        // 原值为null，新值则为传入的value，不为null，则使用function计算，得到新值
        V newValue = (oldValue == null) ? value :
                   remappingFunction.apply(oldValue, value);
        // 新值如果为null，则移除原key-value对
        if(newValue == null) {
            remove(key);
        } else {
            // 不为null，则替换之
            put(key, newValue);
        }
        // 返回新值
        return newValue;
    }
```

```java
    @Test
    public void testMerge(){
        // key为2 的值 加上 +add
        map.merge(2," + add",(oldval, newVal) -> oldval + newVal);
        print(map);
    }
```

## Map的putIfAbsent

```java
    default V putIfAbsent(K key, V value) {
        // 得到原值
        V v = get(key);
        // 原值为null，则替换新值
        if (v == null) {
            v = put(key, value);
        }
		// 返回原值
        return v;
    }
```

```java
    @Test
    public void testPutIfAbsent(){
        // key = 10 对应的value不存在， 则用101010 覆盖
        String s1 = map.putIfAbsent(10, "101010");
        System.out.println(s1);
        print(map);
        System.out.println("============================");
        // key = 2 对应的value存在且为summer，返回summer
        String s2 = map.putIfAbsent(2, "2222");
        System.out.println(s2);
        print(map);
    }
```

输出结果

```java
null
x -> 1, y -> hello
x -> 2, y -> summer
x -> 3, y -> day
x -> 4, y -> tqbx
x -> 10, y -> 101010
============================
summer
x -> 1, y -> hello
x -> 2, y -> summer
x -> 3, y -> day
x -> 4, y -> tqbx
x -> 10, y -> 101010
```

## Map的replace相关方法

```java
    @Test
    public void testReplace(){
        //boolean 将指定的 1 -> hello 键值对的value替换为hi
        map.replace(1,"hello","hi");
        print(map);
        System.out.println("============================");
        //V 指定key对应的value替换成新value
        map.replace(2,"天乔巴夏");
        print(map);
        System.out.println("============================");
        //void 对所有的key和value 进行计算，填入value中
        map.replaceAll((k,v)-> k + "-" + v.toUpperCase());
        print(map);
    }
```

输出结果：

```java
x -> 1, y -> hi
x -> 2, y -> summer
x -> 3, y -> day
x -> 4, y -> tqbx
============================
x -> 1, y -> hi
x -> 2, y -> 天乔巴夏
x -> 3, y -> day
x -> 4, y -> tqbx
============================
x -> 1, y -> 1-HI
x -> 2, y -> 2-天乔巴夏
x -> 3, y -> 3-DAY
x -> 4, y -> 4-TQBX
```

