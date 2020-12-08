## 理解Lambda

Lambda表达式可以是一段可以传递的代码，它的核心思想是**将面向对象中的传递数据变成传递行为**，也就是行为参数化，将不同的行为作为参数传入方法。

随着函数式编程思想的引进，Lambda表达式让可以用更加简洁流畅的代码来代替之前冗余的Java代码。

口说无凭，直接上个例子吧。在Java8之前，关于线程代码是这样的：

```java
class Task implements Runnable{

    @Override
    public void run() {
        System.out.println("Java8 之前 实现Runnable接口中的run方法");
    }
}
Runnable t = new Task();
```

我们定义了一个Task类，让它实现Runnable接口，实现仅有的run方法，我们希望执行的线程体虽然只有一句话，但我们仍然花了大量大代码去定义。为了简化，我们可以采用匿名内部类的方式：

```java
Runnable taskBeforeJava8 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Java8 之前的写法, 传入匿名类");
    }
};
```

但是，其实还是不够简洁，我们用Lambda的写法是这样的：

```java
// java8 之后
Runnable taskAfterJava8 = () -> System.out.println("Java8 之后的写法,lambda表达式");
```

我们仅仅使用`()`和`->`就完成了这件事，是不是非常简洁呢？如果你觉得虽然Lambda写法简洁，但是它的规则让人摸不着头脑，那就跟着我接下去学叭。

## 基础语法

```java
(parameters) -> action
    (parameters) -> expression
    (parameters) -> {statements;}
```

parameters代表变量，可以为空，可以为单，可以为空，你能想到的方式他都可以。

action是实现的代码逻辑部分，可以是一行代码`expression`，也可以是一个代码片段`statements`。如果是代码片段，需要加上`{}`。

下面是一些合法的示例，你可以看看有没有掌握：

| 表达式                    | 描述                              |
| ------------------------- | --------------------------------- |
| `() -> 1024`              | 不需要参数，返回值为1024          |
| `x -> 2 * x`              | 接收参数x，返回其两倍             |
| `(x, y) -> x - y`         | 接收两个参数，返回它们的差        |
| `(int x, int y) -> x + y` | 接收两个int类型参数，返回他们的和 |
| `(String s) -> print(s)`  | 接收一个String对象，并打印        |

## 函数式接口

什么是函数式接口？**函数式接口是只有一个抽象方法的接口**，用作lambda表达式的类型。

```java
@FunctionalInterface // 此注解作用的接口 只能拥有一个抽象方法
public interface Runnable {
    public abstract void run();
}
```

在这里，`@FunctionalInterface`注解是非必须的，有点类似于`@Override`，起强调作用，如果你的接口标注该注解，却没有遵循它的原则，编译器会提示你修改。

### 常用的函数式接口

JDK原生为我们提供了一些常用的函数式编程接口，让我们在使用他们编程时，不必关心接口名，方法名，参数名，只需关注它的参数类型，参数个数，返回值。

| 接口      | 参数 | 返回值  | 类别       | 示例                   |
| --------- | ---- | ------- | ---------- | ---------------------- |
| Consumer  | T    | void    | 消费型接口 | 打印输出某个值         |
| Supplier  | None | T       | 供给型接口 | 工厂方法获取一个对象   |
| Function  | T    | R       | 函数型接口 | 获取传入列表的总和     |
| Predicate | T    | boolean | 断言型接口 | 判断是否以summer为前缀 |

### 消费型接口

```java
    /**
     * 消费型接口, 传入T 无返回值
     */
    public static void consumerTest() {
        Consumer<Integer> consumer = num -> System.out.println(num * num);
        consumer.accept(10);
    }
```

### 供给型接口

```java
    /**
     * 供给型接口, 无参数,返回T
     */
    public static void supplierTest() {
        Supplier<Object> supplier = () -> new Object();
        System.out.println(supplier.get());
    }
```

### 断言型接口

```java
    /**
     * 断言型 传入参数T ,返回boolean
     */
    public static void predicateTest() {
        Predicate<String> predicate = name -> name.startsWith("summer");
        System.out.println(predicate.test("summerday"));
    }
```

### 函数型接口

```java
    /**
     * 函数型接口 传入T 返回R
     */
    public static void functionTest() {
        List<Integer> list = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Function<List<Integer>, Integer> function = num -> {
            int res = 0;
            for (int n : list) {
                res += n;
            }
            return res;
        };
        Integer num = function.apply(list);
        System.out.println(num);
    }
```

## 方法引用

方法引用可以看作特定Lambda表达式的快捷写法，主要分为以下两种：

- 指向静态方法的方法引用
- 指向现有对象的实例方法的方法引用

```java
/**
 * 方法引用
 * 1. 指向静态方法的方法引用
 * 2. 指向现有对象的实例方法的方法引用
 *
 * @author Summerday
 */
public class MethodReferenceTest {
	
    public static List<String> getList(List<String> params, Predicate<String> filter) {
        List<String> res = new LinkedList<>();
        for (String param : params) {
            if (filter.test(param)) {
                res.add(param);
            }
        }
        return res;
    }
    // 静态方法
    public static boolean isStartWith(String name) {
        return name.startsWith("sum");
    }

    public static void main(String[] args) {
        List<String> params = Arrays.asList("summerday","tqbx","天乔巴夏","summer","");
        //静态方法的方法引用 getList(params, name -> MethodReferenceTest.isStartWith(name));
        List<String> list = getList(params, MethodReferenceTest::isStartWith);
        System.out.println(list);
        
        // 指向现有对象的实例方法的方法引用 getList(params, name -> name.isEmpty());
        List<String> sum = getList(params, String::isEmpty);
        System.out.println(sum);
        
    }
}
```

## 数组引用

```java
/**
 * 数组引用
 * @author Summerday
 */
public class ArrayReferenceTest {

    public static void main(String[] args) {
        // 普通lambda
        Function<Integer,String[]> fun1 = x -> new String[x];
        String[] res1 = fun1.apply(10);
        System.out.println(res1.length);

        // 数组引用写法
        Function<Integer,String[]> fun2 = String[]::new;
        String[] res2 = fun2.apply(10);
        System.out.println(res2.length);

    }
}
```

## 构造器引用

```java
/**
 * 构造器引用
 * @author Summerday
 */
public class ConstructorReferenceTest {

    public static void main(String[] args) {

        // 普通lambda
        Supplier<User> sup = () -> new User();

        // 构造器引用
        Supplier<User> supplier = User::new;
        User user = supplier.get();
        System.out.println(user);
    }

}

class User{

}
```

## 总结

1. lambda表达式没有名称，但有参数列表，函数主体，返回类型，可能还有一个可以抛出的异常的列表。
2. lamda表达式让你可以将不同的行为作为参数传入方法。
3. 函数式接口是仅仅声明了一个抽象方法的接口。
4. 只有在接受函数式接口的地方才可以使用lambda表达式。
5. lambda表达式允许你直接内联，为函数式接口的抽象方法提供实现，并将整个表达式作为函数式接口的一个实例。
6. Java8自带了一些常用的函数式接口，包括`Predicate,Function,Supplier,Consumer,BinaryOperator`。
7. 方法引用让你重复使用现有的方法实现并直接传递他们：`Classname::method`。

## 参考阅读

- [跟上Java8 - 了解lambda](https://zhuanlan.zhihu.com/p/28093333)
- [跟上Java8知乎专栏](https://www.zhihu.com/column/java8)