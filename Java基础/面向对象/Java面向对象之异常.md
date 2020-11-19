[toc]
# Java面向对象之异常【一】

终于完成本学期的最后一门考试，考试周的我，边复习通信之傅里叶变换，边学习Java的新知识。虽然很久没更，但是私底下的笔记满满，特地总结一波。
总结什么呢？异常！嗯？异常？最近倒是人有些异常……复习到一两点，早上早早起来刷题，不异常才怪。异常嘛，很好理解，就是本该正常的事情出了问题嘛。
这时我们回想我们之前写过一个再简单不过的例子：计算两个整数相除。二话不说，直接就可以写出如下代码，对吧。

```java
public static void main(String[] args) {
    Scanner input = new Scanner(System.in);
    System.out.println("Enter two integers: ");
    int num1 = input.nextInt();
    int num2 = input.nextInt();
    System.out.println(num1 / num2);
}
```
这短短的代码中就存在着隐患，而这个隐患有可能就会演变成异常。什么情况呢？我们知道，在Java的整数计算中，如果除数为零，计算式将会是没有任何意义的。带着我们的猜想，编译运行，不出所料，它飘红了。
![7229c5f38e87c9abeacccef7d82c708a.png](en-resource://database/6223:1)
于是，我们对红字进行分析，得出结论：一个名为ArthmeticException的异常在执行main方法的第17行时发生：除数为零。清清楚楚，明明白白。
可以看到，异常一旦发生，程序就将终止，而且这种抛出异常的机制，能够让我们有效地找到问题所在，并及时解决问题。异常机制的存在，就是为了更好地解决问题，保证程序的健壮性。
所以，我们试图修改代码，让它在方法中实现：
```java
public static int quotient(int num1, int num2) throws ArithmeticException{
    if (num2 == 0) {
        throw new ArithmeticException("Divisor can not be zero");
    }
    return num1 / num2;
}
```
在main方法中调用查看结果：
```java
int num1 = input.nextInt();
int num2 = input.nextInt();
try {
    System.out.println(QuotientWithMethod.quotient(num1, num2));
}catch (ArithmeticException e) {
    System.out.println(e.getMessage());
}
```
继续测试（当然为了测试，我们这里抛出了一个运行时异常，本可以不抛）：
![131e2d9ee470ad07420de8b25fb400db.png](en-resource://database/6329:1)
- 在求两数之商的方法中主动抛出（throw）一个异常，`throw new ArithmeticException("Divisor can not be zero");`对象。
- 在方法定义处声明方法将会抛出的异常类型，`throws ArithmeticException`。
- 在调用该方法时捕获方法，`try{...}catch (ArithmeticException e){...}`并执行我们希望处理异常的语句，`System.out.println(e.getMessage());`。

当然，上述的例子集抛出异常、声明异常及处理异常于一身，但是异常的内容可不仅仅是这么简单，我们在之后的内容中来探一探异常的究竟：
## 异常的继承体系
先来看一下异常的类继承图，当是不完全统计的，因为还有还多好多的异常等待着被探索。
![734466aefcf6d84b6e8ddc1cec93a019.png](en-resource://database/5998:1)
通过图片我们可以明显的发现，这些异常的命名非常好认，果真见名知义。`Throwable`是所有异常的顶级父类，在它的下面有两个大类：`Exception`和`Error`。下面是官方文档对两者进行的解释：
###  Error
- <u>合理的应用程序出现的不应该捕获的严重问题</u> ，合理指的是语法和逻辑上。

- 不能处理，不要尝试用**catch捕获**，只能尽量优化。
- 它及它的子类异常不需要在**throws语句**中声明，不需要说明他们将会被抛出。
- 它们属于**不受检异常**（unchecked exceptions）。
- 与虚拟机相关，如系统崩溃、虚拟机错误、动态链接失败等



### Exception
- 与**Erro**r不同的是，**Exception**代表的是一类<u>在合理的应用程序中出现的可以处理的问题。</u>
- 除了特例**RuntimeException**及其子类之外，**Exception**的其他异常子类都是**受检异常**（checked exceptions）。

## 异常是否受检

### unchecked exceptions（不受检异常）
也叫**运行时异常**，由Java虚拟机抛出，**只是允许编译时不检测**，在没有捕获或者声明的情况下也一样能够通过编译器的语法检测，所以不需要去亲自捕获或者声明，当然要抛出该类异常也是可以的。典型的异常类型：**Error及其子类异常**，**RuntimeException及其子类异常**。注意：RuntimeException代表的是一类编程错误引发的异常：算数异常、空指针异常、索引越界异常、非法参数异常等等，这些错误如果代码编写方面没有任何漏洞，是完全可以避免的，这也是不需要捕获或者声明的原因，也**有助于简化代码逻辑**。
### checked expections（受检异常)
也叫**编译时异常**，Java认为受检异常需要在编译阶段进行处理， 必须在显式地在调用可能出现异常的方法时捕获（catch），或者在声明方法时**throws**异常类型，否则编译不会通过。除了上面提到的Exception及其子类都属于受检异常，当然，不包括上面提到的**RuntimeException**。

## 异常的处理方式
我们上面提到，我们无法去直接处理不受检异常，但是我们必须强制地对可能发生受检异常（**编译时异常**）的行为做处理，处理方法主要有以下：

- 在当前方法中不知道如何处理该异常时，直接在方法签名中throws该异常类型。
```java
    public void m1() throws ClassNotFoundException, IOException {
        //to do something
    }
```
- 在当前方法中明确知道如何处理该异常，就使用`try{...}catch{...}`语句捕获，并在**catch块**中处理该异常。
```java
public void m3() {
        try {
            m1();
        } catch (ClassNotFoundException e) {
            //to do something
        } catch (IOException e) {
            //to do something
        }
    }
    
```
也可以直接捕获Exception的实例，因为它是这些异常的父类，但是上面的捕获异常引发的错误会更加直接一些。
```java
    public void m2() {
        try {
            m1();
        } catch (Exception e){
            //do something
        }
    }
```
**如果处理不了，就一直向上抛，直到有完善解决的办法出现为止**。当然我们也可以自行抛出系统已经定义的异常：
```java

    public void m2(int i) throws IOException, ClassNotFoundException {
        if (i >1)
            throw new IOException("!");
        throw new ClassCastException();
    }
```
需要注意的是：
- 用throw语句抛出**异常的实例**！注意是异常的实例！且一次只能抛出一个异常！
- throw和throws是不一样的！睁大眼睛！一个是抛出异常实例，一个是声明异常类型。
- 创建异常实例可以传入字符串参数，将被显示异常信息。

## 自定义异常
我们知道，程序发生错误时，系统会自动抛出异常。那么，我们如果想独家定制一个属于自己的异常可以不？答案显然是可以的，情人节快到了，直接自定义一个异常：
```java
    class noGirlFriendException extends Exception{
        noGirlFriendException(){
        }
        noGirlFriendException(String msg){
            super(msg);
        }
    }
```
需要注意的是：
- 如果自定义异常继承RuntimeException，那么该异常就是运行时异常，不需要显式声明类型或者捕获。
- 如果继承Exception，那么就是编译时异常，需要处理。
- 定义异常通常需要提供两个构造器，**含参构造器传入字符串，作为异常对象的描述信息**。


## 异常的捕获方式
- 每个异常**分别对应一个catch语句**捕获。
- 对所有异常处理方式相同，用父类接收，向上转型,Exception对应的catch块应该放在最后面，不然的话，在他后面的块没有机会进入处理。下面就是一个典型的错误示范：
```java
//下面的形式错误！
public static void main(String[] args) {
    try {
        System.out.println(1 / 0);
    } catch (Exception e) {
        e.printStackTrace();
    } catch (ArithmeticException e) {
        System.out.println(e.getMessage());
    }
}
```
- JDK1.7之后，可以分组捕获异常。
    - 异常类型用竖线分开。
    - 异常变量被隐式地被final修饰。
```java
try {
    System.out.println(1 / 0);
} catch (ArithmeticException | NullPointerException e) {
    //捕获多异常时，异常变量被final隐式修饰
    //！false: e = new ArithmeticException();
    System.out.println(e.getMessage());
} 
```
本文参考诸多资料，并加上自身理解，如有叙述不当之处，还望评论区批评指正。关于异常，还有一部分内容，下篇进行总结，晚安。
参考资料：
[https://stackoverflow.com/questions/6115896/understanding-checked-vs-unchecked-exceptions-in-java](
https://stackoverflow.com/questions/6115896/understanding-checked-vs-unchecked-exceptions-in-java)

[https://www.programcreek.com/2009/02/diagram-for-hierarchy-of-exception-classes/](
https://www.programcreek.com/2009/02/diagram-for-hierarchy-of-exception-classes/)

[https://stackoverflow.com/search?q=%5BJava%5D+Exception](
https://stackoverflow.com/search?q=%5BJava%5D+Exception)

# Java面向对象之异常【二】
> 往期回顾：上一篇我们大致总结了异常的继承体系，说明了Exception和Error两个大类都继承于顶级父类Throwable，又谈到编译时异常与运行时异常的区别，谈到异常的处理方式，以及处理方式中关于捕获方式的几种类型。
> 本篇承上启下，将从异常的其余部分进行总结，但是毕竟现在处于初学阶段，未必能够体会异常在真实场景中运用的便利之处，所以本文只是对目前所学内容的归纳整理，后续新的体会将会及时更新。
## 捕获异常的规则
- 在执行try块的过程中没有出现异常，那么很明显，**没有异常当然就会跳过catch子句**。
- 相反，如果抛出了一个异常，那么就会**跳过try中的剩余语句**，开始查找处理该异常的代码。以下是查找处理异常代码的具体过程：
>  - 从当前方法开始，沿着方法的调用链，按照异常的反向传播方向找到异常的处理代码。
>   - 从第一个到最后一个检查catch块，判断是否相匹配。如果是，那么恭喜！直接进入catch块中执行处理异常语句；如果不是，就将该异常传给方法的调用者，在调用者中继续执行相同步骤：匹配就处理，不匹配就向上传……
>   - 直到最后如果都没有找到的话，程序将会终止，并在打印台上打印出错信息。
>  如果是相对单一方法而言，其实是很简单的；如果方法层层嵌套呢，情况又是咋样的呢，咱们来验证一下以上内容：
```java
//主方法
public static void main(String[] args) {
    try{
    //调用m1()
        m1();
        System.out.println("ExceptionMyDemo.main");
    }catch (Exception e){
        System.out.println("ExceptionMyDemo.main.catch");
    }
}
//m1()
private static void m1(){
    try{
    //调用m2()
        m2();
        System.out.println("ExceptionMyDemo.m1");
    }catch (NullPointerException e){
        System.out.println("ExceptionMyDemo.m1.catch");
    }
}
//m2()
private static void m2(){
    String str = null;
    System.out.println(str.hashCode());
}
//测试结果：
ExceptionMyDemo.m1.catch
ExceptionMyDemo.main
```
- 可以看到，m1中捕获了m2抛出匹配的空指针异常类型，直接处理，在main方法中就接收不到异常，也就正常执行。
- 假如我们把m1的catch的异常类型换成其他的类型，比如`catch (ArithmeticException e)`，这时的测试结果会是这个样子：
```java
//更改之后的测试结果：
ExceptionMyDemo.main.catch
```
因为m2()抛出的异常在m1中并没有被合适地处理，所以向上抛出，在main方法中找到了处理方法，遂执行处理语句。
- 依据我们上面所说，如果在上面更改之后的基础上，再把main方法中的处理异常语句删去，那么程序运行的结果会是啥呢？哦，不出所料，是下面的结果：
![d5d7d0df6396bf0c3995bdf1be70fcf7.png](en-resource://database/6333:1)

因为抛出的异常没人处理，它就会在控制台上打印异常的栈轨迹，关于抛出的异常信息，我们接下来进行详细分析。
## 访问异常信息
我们提到，无论是虚拟机抛出异常还是我们主动抛出，异常的错误信息都包含其中，以便于我们得知并更好地处理异常，那么顺着上面所说，我们刚刚看到的就是异常的栈轨迹：


![4725225c87dcba438cb01a3d191bc3d1.png](en-resource://database/6337:1)


- `public void printStackTrace()`：默认将该Throwable对象及其调用栈的跟踪信息打印到标准错误流。
- `public String getMessage()`：返回描述异常对象信息的字符串。
- `public String toString()`:异常信息message为空就返回异常类的全名，否则返回`全名：message`的形式。
- `public StackTraceElement[] getStackTrace()`：返回栈跟踪元素的数组，表示和该异常对象相关的栈的跟踪信息。

## 异常对方法重写的影响
- 异常对方法重载没有影响。
- 方法重写时，**子类中重写的方法抛出的编译时异常不能超过父类方法抛出编译时异常的范围**。

## finally详解
名言警句：**无论异常是否会发生，finally修饰的子句总是会被执行**。

于是我们进行了简单的尝试：

```java
public static void m2(){
    try{
        System.out.println("0");
        System.out.println(1/0);
        System.out.println("1");
    }
    catch (Exception e){
        System.out.println("2");
    }
    finally {
        System.out.println("3");
    }
    System.out.println("4");
}
//测试结果
0 2 3 4 被打印在控制台上
```
- 可以看到1没有被打印，因为在执行` System.out.println(1/0);`时发生了异常，于是进入catch块，finally子句必会被执行，然后执行try语句后的下一条语句。
- 想象以下：假如把接收异常的实例类型改为另外一个不匹配的类型的话，也就是说无法正常捕获，结果又会如何呢？结果如下：
![25b131a70efce43bb505fdfa92bafd7f.png](en-resource://database/6339:1)
- 很明显，这时候finally的效果就出来了，就算你出了异常，我finally块中的语句必须要执行，这个在现实场景中对于**释放资源**起了很关键的作用，但是具体来说，由于还没有学习后面的内容，就暂且不提了，有些东西还是体会之后会更加真实一些。
- 还有一个注意点就是4也没有被打印出来，是因为没有捕获到异常，将会把异常抛给调用者，所以不会执行`System.out.println("4");`。

但是，化名为几千万个为什么的我又开始疑惑了，我们直到return可以将方法直接返回，强制退出。那么如果在try中使用return语句，finally还会不会不忘初心，继续执行呢？

前方高能！各单位注意！！！
猜猜看，这四个方法执行结果是啥呢？
```java
private  static int m1(){
    try{
        return 1;
    }catch(Exception e){
    }
    return 2;
}
```
```java
private static int m2(){
    try{
        return 1;
    }finally {
        return 2;
    }
    //使用finally子句时可以省略catch块
}
```
```java
private static int m3(){
    try{
        return 1;
    }finally {
        try{
            return 2;
        }finally {
            return 3;
        }
    }
}
```
```java
private static int m4(){
    int i = 4;
    try{
        return i++;
    }finally {
        i++;
    }
}
```
答案揭晓：分别是：1，2，3，4。你们猜对了吗？哈哈……

我想前三个答案应该是毋庸置疑的，但是这第四个就有点离谱了。不是说finally语句一定会执行吗，执行哪去了呢，你要是执行的话，你i难道不应该变成6了吗？
额……咳咳，这个嘛，我也有点迷惑，但是经过一番讨教，稍微懂了一些：
- 当执行try之前，如果后面有finally，会将try中的返回过程延迟，就是说把i=4放到结果区。
- 然后在计算区进行自增运算变为5，finally语句一定会执行，但是只是在计算区域自增为6了，结果区域还是原来的那个4。
- 不信的话，你可以在finally语句的i++后面看看i的值，它！就是6！所以说finally子句一定执行是毋庸置疑的的！

但是如果进行改变的是引用数据类型的变量时，那么就会随之改变了，人家村的是地址，改的就是本身。我在这边就稍微来个简单的例子奥：
```java
public static Student m(){
    Student s = new Student();
    try{
        s.age = 20;
        s.name = "天乔";
        return s;
    }finally {

        s.name = "巴夏";
        s.age = 2;
    }
}
//测试结果
//Student{age=2, name='巴夏'}
```