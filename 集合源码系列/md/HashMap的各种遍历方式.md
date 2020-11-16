[toc]

## 前言

- 介绍HashMap遍历的几种方式
- 介绍HashMap迭代删除的几种方式

## HashMap遍历的几种方式

### 一、迭代器遍历

#### 迭代EntrySet

```java
    @Test
    public void testEntrySet() {
        Iterator<Map.Entry<String, String>> iterator = map.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, String> entry = iterator.next();
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        }
    }
```

#### 迭代KeySet

```java
    @Test
    public void testKeySet() {
        Iterator<String> iterator = map.keySet().iterator();
        while (iterator.hasNext()) {
            String key = iterator.next();
            System.out.println(key + " --> " + map.get(key));
        }
    }
```

### 二、ForEach遍历

#### 遍历EntrySet

```java
    @Test
    public void testForEachEntrySet() {
        for (Map.Entry<String, String> entry : map.entrySet()) {
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        }
    }
```

#### 遍历KeySet

```java
    @Test
    public void testForEachKeySet() {
        for (String key : map.keySet()) {
            System.out.println(key + " --> " + map.get(key));
        }
    }
```

### 三、Lambda遍历

```java
    @Test
    public void testLambda() {
        map.forEach((key, value) -> {
            System.out.println(key + " --> " + value);
        });
    }
```

### 四、StreamAPI遍历

```java
    @Test
    public void testStream() {
        map.entrySet().stream().forEach(entry -> {
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        });
    }
```

```java
    @Test
    public void testParallelStream() {
        map.entrySet().parallelStream().forEach(entry -> {
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        });
    }
```

## 各种遍历方式的性能比较

引入jmh性能测试框架

```xml
    <dependencies>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.23</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>1.20</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
```

编写测试

```java
@BenchmarkMode(Mode.Throughput) //测试类型 吞吐量
@Warmup(iterations = 2, time = 1, timeUnit = TimeUnit.SECONDS) //预热2轮,每次1秒
@Measurement(iterations = 5, time = 3, timeUnit = TimeUnit.SECONDS) //测试5轮,每次3s
@Fork(1)  //fork 1个线程
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Thread) //每个测试线程1个实例
public class HashMapBenchmark {
    static Map<String, String> map = new HashMap<String,String>() {
        {
            put("author", "天乔巴夏");
            put("title", "HashMap的各种遍历方式");
            put("url", "www.hyhwky.com");
        }
    };

    public static void main(String[] args) throws RunnerException {
        System.out.println(System.getProperties());
        Options opt = new OptionsBuilder()
                .include(HashMapBenchmark.class.getSimpleName())
                .output(System.getProperty("user.dir") + "\\HashMapBenchmark.log")
                .build();
        new Runner(opt).run();
    }

    @Benchmark
    public void testEntrySet() {
        Iterator<Map.Entry<String, String>> iterator = map.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<String, String> entry = iterator.next();
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        }
    }

    @Benchmark
    public void testKeySet() {
        Iterator<String> iterator = map.keySet().iterator();
        while (iterator.hasNext()) {
            String key = iterator.next();
            System.out.println(key + " --> " + map.get(key));
        }
    }

    @Benchmark
    public void testForEachEntrySet() {
        for (Map.Entry<String, String> entry : map.entrySet()) {
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        }
    }

    @Benchmark
    public void testForEachKeySet() {
        for (String key : map.keySet()) {
            System.out.println(key + " --> " + map.get(key));
        }
    }

    @Benchmark
    public void testLambda() {
        map.forEach((key, value) -> {
            System.out.println(key + " --> " + value);
        });
    }

    @Benchmark
    public void testStream(){
        map.entrySet().stream().forEach(entry ->{
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        });
    }

    @Benchmark
    public void testParallelStream(){
        map.entrySet().parallelStream().forEach(entry ->{
            System.out.println(entry.getKey() + " --> " + entry.getValue());
        });
    }
}

```

测试结果如下

```java
Benchmark                              Mode  Cnt  Score   Error   Units
HashMapBenchmark.testEntrySet         thrpt    5  6.929 ± 1.042  ops/ms
HashMapBenchmark.testForEachEntrySet  thrpt    5  7.025 ± 0.627  ops/ms
HashMapBenchmark.testForEachKeySet    thrpt    5  7.024 ± 0.481  ops/ms
HashMapBenchmark.testKeySet           thrpt    5  6.769 ± 1.231  ops/ms
HashMapBenchmark.testLambda           thrpt    5  7.056 ± 0.300  ops/ms
HashMapBenchmark.testParallelStream   thrpt    5  5.784 ± 0.216  ops/ms
HashMapBenchmark.testStream           thrpt    5  6.826 ± 0.578  ops/ms
```

- Score表示平均执行时间，Error表示误差。
- 测试结果和本机性能有一定关系，只需关注相对结果即可。
- 可以看出除了Parallel的遍历方法，其他方法在性能上的差别不大。

## HashMap迭代删除的几种方式

### 迭代器删除

```java
    @Test
    public void testRemoveIterator() {
        Iterator<Map.Entry<String, String>> iterator = map.entrySet().iterator();
        while (iterator.hasNext()) {
            String key = iterator.next().getKey();
            if ("title".equals(key)) {
                iterator.remove();
            }
        }
        System.out.println(map);
    }
```

### Lambda的removeIf

```java
    @Test
    public void testRemoveLambda() {
        map.keySet().removeIf("title"::equals);
        System.out.println(map);
    }
```

### StreamAPI的filter

```java
    @Test
    public void testRemoveStream() {
        map.entrySet()
                .stream()
                .filter(m -> !"title".equals(m.getKey()))
                .forEach(entry -> {
                    System.out.println(entry.getKey() + " --> " + entry.getValue());
                });
    }
```

## 参考文章

- Java中文社群：你一般是怎么遍历HashMap的？

