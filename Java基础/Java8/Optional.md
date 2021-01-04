[toc]

## Optional概述

Optional 是个容器：它可以保存类型T的value，或者仅仅保存null。Optional提供很多有用的方法，这样我们就不用显式进行空值检测，很好地解决了空指针异常处理的问题，比如可以使用`isPresent()`方法判断value是否为null，使用`get()`方法获取value值等等。

Optional的构造方法是私有的，实例不能new，可以使用静态方法来创建：

```java
// 1、创建一个包装对象值为空的Optional对象
Optional<String> optStr = Optional.empty();
// 2、创建包装对象值非空的Optional对象
Optional<String> optStr1 = Optional.of("optional");
// 3、创建包装对象值允许为空的Optional对象
Optional<String> optStr2 = Optional.ofNullable(null);
```

## Optional简单案例

看完Optional的概述，我们用一个简单的例子说明一下：

下面这段代码接收一个User对象，如果user为null，则抛出异常【这是一个非常常规的避免空指针异常的做法，如果没有这步，getName会NPE】，否则返回user的name。

```java
    public String getName1(User user) {
        if (user == null) {
            throw new RuntimeException("user不能为null!");
        }
        return user.getName();
    }
```

如果使用Optional，应该怎么去处理呢？

```java
    public String getName2(User user) {
        return Optional.ofNullable(user) // 包装user对象，如果user为null，则返回空的Optional对象
                .map(User::getName)
                .orElseThrow(() -> new RuntimeException("user不能为null"));// 如果有值则返回，没有则抛出异常。
    }
```

- Optional使用静态的`ofNullable`方法将user对象进行包装，将可能为null的user对象保护起来。

```java
    public static <T> Optional<T> ofNullable(T value) {
        // empty() 方法 创建一个空的Optional对象， of对象在构造Optional的时候，value如果weinull，会引发NPE
        return value == null ? empty() : of(value);
    }
```

- orElseThrow方法接收一个Supplier 对象，这里我们在lambda表达式那节已经说过了，不再赘述，感兴趣可以查看：[Java8的Lambda表达式，你会不？](https://blog.csdn.net/Sky_QiaoBa_Sum/article/details/110789826)

```java
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        // 如果有值，直接返回值
        if (value != null) {
            return value;
        } else {
            // 抛出异常，这个异常Supplier接口定义
            throw exceptionSupplier.get();
        }
    }
```

## Optional的主要方法

| 方法          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `empty`       | 返回一个空的 Optional 实例                                   |
| `filter`      | 如果值存在并且满足提供的断言， 就返回包含该值的 Optional 对象；否则返回一个空的 Optional 对象 |
| `map`         | 如果值存在，就对该值执行提供的 mapping 函数调用              |
| `flatMap`     | 如果值存在，就对该值执行提供的 mapping 函数调用，返回一个 Optional 类型的值，否则就返 回一个空的 Optional 对象 |
| `get`         | 如果该值存在，将该值用 Optional 封装返回，否则抛出一个 NoSuchElementException 异常 |
| `ifPresent`   | 如果值存在，就执行使用该值的方法调用，否则什么也不做         |
| `isPresent`   | 如果值存在就返回 true，否则返回 false                        |
| `of`          | 将指定值用 Optional 封装之后返回，如果该值为 null，则抛出一个 NullPointerException 异常 |
| `ofNullable`  | 将指定值用 Optional 封装之后返回，如果该值为 null，则返回一个空的 Optional 对象 |
| `orElse`      | 如果有值则将其返回，否则返回一个默认值                       |
| `orElseGet`   | 如果有值则将其返回，否则返回一个由指定的 Supplier 接口生成的值 |
| `orElseThrow` | 如果有值则将其返回，否则抛出一个由指定的 Supplier 接口生成的异常 |

## 参考阅读

- [Java 8 Optional 类](https://www.runoob.com/java/java8-optional-class.html)
- [【java8新特性 简述】Optional](https://www.jianshu.com/p/0fd4637a55a6)
- [https://github.com/biezhi/learn-java8/blob/master/java8-optional/README.md](https://github.com/biezhi/learn-java8/blob/master/java8-optional/README.md)