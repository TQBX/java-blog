## Callable

Callable接口的定义如下：

```java
@FunctionalInterface
public interface Callable<V> {
	// 计算结果
    V call() throws Exception;
}
```

可以看到这个接口中只有一个抽象方法call，因此可以传入Java8支持的Lambda表达式。另外，这是一个泛型接口，call方法返回类型V，

## Future





## FutureTask

FutureTask接口实现了RunnableFuture接口：

```java
public class FutureTask<V> implements RunnableFuture<V>
```

RunnableFuture接口又实现了Runnable和Future接口：

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {

    void run();
}
```

懂了吧，

http://ifeve.com/executorservice-10-tips-and-tricks/

https://www.cnblogs.com/dolphin0520/p/3949310.html

