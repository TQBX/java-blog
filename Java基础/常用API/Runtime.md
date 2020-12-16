## Runtime类简介

Java中，Runtime类提供了许多的API 来与`java runtime environment`进行交互，如：

- 执行一个进程。
- 调用垃圾回收。
- 查看总内存和剩余内存。

Runtime是单例的，可以通过`Runtime.getRuntime()`得到这个单例。

## API列表

| public static Runtime getRuntime()                    | 返回单例的Runtime实例                           |
| :---------------------------------------------------- | :---------------------------------------------- |
| public void exit(int status)                          | 终止当前的虚拟机                                |
| public void addShutdownHook(Thread hook)              | 增加一个JVM关闭后的钩子                         |
| public Process exec(String command)throws IOException | 执行command指令，启动一个新的进程               |
| public int availableProcessors()                      | 获得JVM可用的处理器数量（一般为CPU核心数）      |
| public long freeMemory()                              | 获得JVM已经从系统中获取到的总共的内存数【byte】 |
| public long totalMemory()                             | 获得JVM中剩余的内存数【byte】                   |
| public long maxMemory()                               | 获得JVM中可以从系统中获取的最大的内存数【byte】 |

注：以上为列举的比较常见的一些方法，不完全。

## 经典案例

### exec

```java
    @Test
    public void testExec(){
        Process process = null;
        try {
            // 打开记事本
            process = Runtime.getRuntime().exec("notepad");
            Thread.sleep(2000);
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }finally {
            if(process != null){
                process.destroy();
            }
        }
    }
```

### 获取信息

```java
    @Test
    public void testFreeMemory(){
        Runtime r = Runtime.getRuntime();
        System.out.println("处理器个数: " + r.availableProcessors());
        System.out.println("最大内存 : " + r.maxMemory());
        System.out.println("总内存 : " + r.totalMemory());
        System.out.println("剩余内存: " + r.freeMemory());
        System.out.println("最大可用内存: " + getUsableMemory());

        for(int i = 0; i < 10000; i ++){
            new Object();
        }

        System.out.println("创建10000个实例之后, 剩余内存: " + r.freeMemory());
        r.gc();
        System.out.println("gc之后, 剩余内存: " + r.freeMemory());

    }
    /**
     * 获得JVM最大可用内存 = 最大内存-总内存+剩余内存
     */
    private long getUsableMemory() {
        Runtime r = Runtime.getRuntime();
        return r.maxMemory() - r.totalMemory() + r.freeMemory();
    }
```

```java
处理器个数: 4
最大内存 : 3787980800
总内存 : 255328256
剩余内存: 245988344
最大可用内存: 3778640888
创建10000个实例之后, 剩余内存: 244650952
gc之后, 剩余内存: 252594608
```

### 注册钩子线程

```java
    @Test
    public void testAddShutdownHook() throws InterruptedException {
        Runtime.getRuntime().addShutdownHook(new Thread(()-> System.out.println("programming exit!")));
        System.out.println("sleep 3s");
        Thread.sleep(3000);
        System.out.println("over");
    }
```

3s之后，test方法结束，打印信息。

### 取消注册钩子线程

```java
    @Test
    public void testRemoveShutdownHook() throws InterruptedException {
        Thread thread = new Thread(()-> System.out.println("programming exit!"));
        Runtime.getRuntime().addShutdownHook(thread);
        System.out.println("sleep 3s");
        Thread.sleep(3000);
        Runtime.getRuntime().removeShutdownHook(thread);
        System.out.println("over");
    }
```

### 终止！

```java
    @Test
    public void testExit(){
        Runtime.getRuntime().exit(0);
        //下面这段 无事发生
        System.out.println("Program Running Check");
    }
```

## 参考阅读

- [https://www.javatpoint.com/java-runtime-class](https://www.javatpoint.com/java-runtime-class)
- [https://www.geeksforgeeks.org/java-lang-runtime-class-in-java/](https://www.geeksforgeeks.org/java-lang-runtime-class-in-java/)