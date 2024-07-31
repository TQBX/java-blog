[toc]

# 面向对象三大特性

封装：对抽象的事物抽象化成一个对象，并对其对象的属性私有化，同时提供一些能被外界访问属性的方法；

继承： 子类扩展新的数据域或功能，并复用父类的属性与功能，单继承，多实现；

> Java和C++的区别，没有多继承（可以用接口实现），不提供指针来直接访问内存，并且有JVM自动内存管理机制，不需要程序员手动释放无用内存（注意内存泄漏）

多态： 通过继承（多个子类对同一方法的重写）、也可以通过接口（实现接口并覆盖接口）

> 多态的实现原理：动态绑定，运行时才把方法调用和方法实现关联起来。

# 重载和重写

重载override指在同一个类中定义多个方法，这些方法名称相同，签名不同。

重写overwrite指在子类中的方法的名称和签名都和父类相同，使用override注解

# this

关键字 this 代表当前对象的引用。当前对象指的是调用类中的属性或方法的对象

关键字 this 不可以在静态方法中使用。静态方法不依赖于类的具体对象的引用

# 构造方法

构造方法可以被重载（不能被重写）

当类中没有显性声明任何构造方法时，才会有默认构造方法。

构造方法没有返回值，构造方法的作用是创建新对象。

# 初始化块

静态初始化块的优先级最高，会最先执行，在非静态初始化块之前执行。

静态初始化块会在类第一次被加载时最先执行，因此在 main 方法之前。

# static和final

## static

修饰方法和属性：

- 方法（随类加载，类名直接调用，方法里面只能使用静态成员，不可以用this）

- 属性（类级别，所有实例共享一份，随类加载（只加载一次），先于对象创建，可以类名直接调）

## final

修饰变量、方法、类：

- 变量：基本数据类型（值不能改） 和 引用数据类型（地址不能改，不能再指向其他对象）
- 方法：锁定方法，防止继承重写；类中的所有private方法都隐式的指定为final
- 类：锁定类无法被继承，同理final类的所有成员方法都是final的（除了final之外，私有化构造起也可以让一个类无法被继承）

# 抽象类和接口

抽象类：包含抽象方法（没有方法体），abstract修饰。无法实例化，只能被继承，因此无法用final修饰，成员变量可以任何修饰符

接口：抽象类型，接口支持多实现，成员变量只能是public static final的，Java8之前方法默认是public abstract修饰的抽象方法，8之后可以定义default和static方法，java9之后可以有private方法（在接口内部共享方法）

使用场景：抽象类定义模版方法，接口定义标准规范，让调用者无需考虑具体实现

# 基本数据类型和包装类

8byte 16short 32int 64long 32float 64double 16char 1boolean (单位：bit)

包装类：缓存范围～Byte Short Integer Long [-128, 127]，Character [0, 127] Boolean[false, true]

为什么要有包装类啊？

- Java中万物皆对象，所以给每一个基本数据类型都配套提供包装类型

自动装箱（基本数据 到 包装类）和拆箱（包装类 到 基本数据）？

- 编译器在底层自动调用valueOf方法，自动装箱
- 编译器在底层自动调用xxxValue方法，自动拆箱

自动装箱和拆箱的问题？

- 超出缓存范围，自动装箱可能会new出很多无用的实例对象！ 

# 泛型以及泛型擦除

泛型：参数化类型，这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口和泛型方法。

**泛型擦除是什么？**

Java的泛型是伪泛型，使用泛型的时候加上类型参数，在编译器编译生成的字节码的时候会去掉，这个过程成为类型擦除

如List等类型，在编译之后都会变成 List。JVM 看到的只是 List，而由泛型附加的类型信息对 JVM 来说是不可见的。可以通过反射添加其它类型元素

# 反射的原理以及使用场景

**什么是发射？**

允许在运行时动态获取内部的成员信息（包括属性、方法和构造函数等）以及动态调用对象的方法

**原理是？**

能够获取到Java中的反射类的字节码，然后将字节码中的方法，变量，构造函数等映射成相应的
Method、Filed、Constructor 等类

得到Class实例的几种方法

```java
1.类名.class(就是一份字节码)
2.Class.forName(String className);根据一个类的全限定名来构建Class对象
3.每一个对象多有getClass()方法:obj.getClass();返回对象的真实类型
```

**场景有哪些？**

1. 开发框架：Spring、mybatis等都是配置化的，保证通用性，运行时根据配置文件动态加载不同的对象或类，调用不同的方法
2. 动态代理：运行时创建代理对象，并在代理对象的方法调用前后执行额外的逻辑（如日志记录、性能监控等）。Spring中有两种代理：JDK动态代理（实现接口的类），CGLIB（通过asm框架序列化字节流）

# Java异常体系

![image-20240724153944173](img/Java%E5%9F%BA%E7%A1%80%E9%83%A8%E5%88%86%E7%9F%A5%E8%AF%86%E7%82%B9%E6%80%BB%E7%BB%93/image-20240724153944173.png)

Throwable：Error（运行时系统内部错误or资源耗尽）  + Exception（Runtime + Checked）

异常处理：try catch finally

# 序列化和反序列化

序列化：将对象的状态 转化为 字节流，对象持久化

反序列化：根据字节流 重建 对象。

优点：实现数据持久化、可以实现远程通信，在网络上传送对象的字节序列

反序列化失败的场景：反序列ID不一致

# String相关问题

## String、StringBuilder和StringBuffer的区别

String 是不可变的 StringBuilder和StringBuffer 都提供了append方法

String不可变，线程安全，StringBuffer加了同步锁也安全，StringBuilder不安全

StringBuilder比String、StringBuffer性能要好

java9之后，都用byte[] 存储子符串，8byte[]相较于16char[]省一半空间

## String不可变的原因？

- 数组是final修饰并且私有，并且String也没有提供修改字符串的方法
- String类被final修饰导致不能被继承

## 字符串常量池的作用

JVM为了提升性能和减少内存消耗，针对字符串专门开辟的一块区域，避免字符串重复创建

## String s1 = new String("abc") 创建几个对象

如果字符串常量池不存在“abc”的引用，会在堆上创建两个字符串对象，其中一个的引用会被包存在字符串常量池

如果字符串常量池已经存在“abc”的引用，只会在堆中创建1个字符串对象。

# Object类相关问题

## 有哪些方法

【toString】默认是指针，一般要重写、【equals】默认是==，String类重写了、【hashCode】重写equal需要重写hashcode、【finalize】垃圾回收前的遗嘱、【clone】深浅拷贝、【getClass】反射获取对象元数据，包括类名，方法、【notify、wait、notifyAll】线程通知和唤醒

## equals的性质

equals有四个性质：

1. 自反性（reflexive）：对于任何非空引用x，x.equals(x)为true。
2. 对称性（symmetric）：对于任何非空的引用x和y，当且仅当y.equals(x)返回true时，x.equals(y)返回true
3. 传递性（transitive）：对于任何非空引用x、y和z，如果x.equals(y)返回true，而y.equals(z)返回true，x.equals(z)也应该返回true。
4. 一致性（consistent）：如果非空引用x和y的对象没有变化，反复调用x.equals(y)返回相同的结果。

## equals和==

==：基本比较值，引用比较内存地址

equals：所有的类都有这个方法，不重写就是==，一般要重写

## equals和hashcode

为什么重写equal时候一定要重写hashcode方法？

如果不改的话，可能导致equal判断A和B相等，但是A和B的hashcode不同，这个equals希望保证的是不符合的！

1. **当equals方法被重写时，应该重写hashCode方法**，从而保证两个相等的对象拥有相同的哈希码。
2. 程序执行过程中，如果对象的数据没有被修改，则多次调用hashCode方法将返回相同的整数。
3. **两个不相等的对象可能具有相同的哈希码**，但在实现hashCode方法时应避免太多这样的情况出现。

## wait

Object wait() 方法让当前线程进入等待状态。直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法。

当前线程必须是此对象的监视器所有者，否则还是会发生 **IllegalMonitorStateException** 异常。

如果当前线程在等待之前或在等待时被任何线程中断，则会抛出 **InterruptedException** 异常。

## wait、sleep、yield比较

当前线程对象调用wait，进入等待状态，释放锁，不会自动苏醒，需要别的线程调用同一对象上的notify或notifyAll。

sleep方法是Thread的静态方法，进入等待，但不释放锁。sleep()方法执行完成后，将会自动苏醒。

调用yield()静态方法，暂时暂停当前线程，让系统的线程调度器重新调度一次，它自己完全有可能再次运行。

# 深拷贝和浅拷贝的区别

> 浅：在堆上创建一个新对象，不过，如果远对象内部属性是引用类型，浅拷贝会直接复制该地址，也就是共用一个内部对象
>
> 深：完全复制，创建新对象（内部引用需要实现cloneable接口）
>
> Beanutils.copyProperties()是浅拷贝

> https://www.jianshu.com/p/94dbef2de298

![shallow&deep-copy](img/Java%E5%9F%BA%E7%A1%80%E9%83%A8%E5%88%86%E7%9F%A5%E8%AF%86%E7%82%B9%E6%80%BB%E7%BB%93/shallow&deep-copy.png)

**浅拷贝**：只复制指向某个对象的指针，而不复制对象本身，新旧对象还是共享同一块内存。

1. 基本类型：一个对象修改值，不会影响另一个。注意：String是final类，改变这个值其实是new一个新的String类型的对象，所以不会影响原对象。
2. 引用类型： 数组或类，内存地址相同，改变其中一个会影响另一个。

浅拷贝的实现： 实现对象拷贝的类，需要实现 `Cloneable` 接口，并覆写 `clone()` 方法。

![img](img/Java基础部分知识点总结/8878793-61eb6dbc8885a9bc.webp)

**深拷贝**：创建一个完全一样的对象，且新老对象不共享内存。

1. 基本类型：显然是不会影响的。
2. 引用类型：既然是深拷贝，为引用类型的数据成员开辟了独立的内存空间，也不会相互影响。
3. 速度慢且花销大。

深拷贝的实现： 深拷贝不仅复制对象本身及其基本类型字段，还递归地复制其引用的所有对象。因此，原始对象和拷贝对象完全独立，互不影响。

![img](img/Java基础部分知识点总结/8878793-858c3f7c993e4348.webp)

# 创建对象有几种方法？

1. new：`Object obj = new Object();`
2. 反射：
   1. 使用Class对象的newInstance，要求Class对象对应的类有默认构造器。
   2. 使用Class对象获取Constructor，调用Construtor对象的newInstance()方法来创建该Class对象对应类的实例。

3. 反序列化

   序列化可以 保存对象状态【持久化】 或  RMI远程方法调用作为网络传输对象。

   ```java
   Person summerday = new Person(); // Person 实现 serializable
   ObjectOutputStream out = 
       new ObjectOutputStream(new FileOutputStream("person.ser"));
   
   out.writeObject(summerday); 
   
   // 下面是反序列化
   ObjectInputStream in = 
       new ObjectInputStream(new FileInputStream("person.ser"));  
   Person summerday = (Person) in.readObject(); 
   ```

4. clone
   1. 浅拷贝直接实现Cloneable接口，重写clone方法。
   2. 深拷贝的话，内层对象也需要cloneable。



# 程序结果输出类题目

## String类

```java
public class Example{
    String str=new String("hello");
    char[]ch={'a','b'};
    public static void main(String args[]){
        Example ex=new Example();
        ex.change(ex.str,ex.ch);
        System.out.print(ex.str+" and ");
        System.out.print(ex.ch);
    }
    public void change(String str,char ch[]){
        str="test ok";
        ch[0]='c';
    }
}
// 正确答案 hello and cb
```

![](img/Java%E5%9F%BA%E7%A1%80%E9%83%A8%E5%88%86%E7%9F%A5%E8%AF%86%E7%82%B9%E6%80%BB%E7%BB%93/string.png)

大致如上图所示，需要注意绿色的Test Ok实在字符串常量池中创建，而不是图中的栈区。

## 类加载相关

```java
package NowCoder;
class Test {
    public static void hello() {
        System.out.println("hello");
    }
}
public class MyApplication {
    public static void main(String[] args) {
        // TODO Auto-generated method stub
        Test test = null;
        test.hello();
    }
}
// 正确结果： 能编译通过，并正确运行
```

关于结果的解释：[https://www.nowcoder.com/profile/710353654/test/38531333/56439#summary](https://www.nowcoder.com/profile/710353654/test/38531333/56439#summary)



# 在创建派生类对象，构造函数的执行顺序

```java
public class ExtendsTest {

    public static void main(String[] args) {
        A c = new  B();
    }

}

class A{
    static
    {
        System.out.println("A 基类静态域 ");
    }
    {
        System.out.println("A 基类对象成员构造函数");
    }
    public A(){
        System.out.println("A 基类本身的构造函数");
    }
}
class B extends A{
    static
    {
        System.out.println("B 派生类静态域");
    }
    {
        System.out.println("B 派生类对象成员构造函数");
    }
    public B(){
        System.out.println("B 派生类本身的构造函数");
    }
}
// 输出结果
A 基类静态域 
B 派生类静态域
A 基类对象成员构造函数
A 基类本身的构造函数
B 派生类对象成员构造函数
B 派生类本身的构造函数
```

# 二维数组的创建

```java
float f[][] = new float[6][6];
float []f[] = new float[6][6];
float [][]f = new float[6][6];
float [][]f = new float[6][];
```

以上都是正确的：

- 行必须指定。
- `[][]`和变量名的顺序可以不定。
