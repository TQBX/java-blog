> Java8新增的Stream + Lambda = ！！！起飞，谁用谁知道！

## 什么是Stream？

`Stream`将要处理的元素集合看作一种流，在流的过程中，借助`Stream API`对流中的元素进行操作，比如：筛选、排序、聚合等。

`Stream`可以由数组或集合创建，对流的操作分为两种：

1. 中间操作，每次返回一个新的流，可以有多个。
2. 终端操作，每个流只能进行一次终端操作，终端操作结束后流无法再次使用。终端操作会产生一个新的集合或值。

另外，`Stream`有几个特性：

1. stream不存储数据，而是按照特定的规则对数据进行计算，一般会输出结果。
2. stream不会改变数据源，通常情况下会产生一个新的集合或一个值。

## Stream的创建

```java
public class BuildStream {

    public static void main(String[] args) {

        List<Integer> list = Arrays.asList(1, 2, 3);
        // 通过集合的stream()方法创建顺序流
        Stream<Integer> stream1 = list.stream();

        // 通过集合的parallelStream()方法创建顺序流
        Stream<Integer> parallelStream1 = list.parallelStream();

        // 通过parallel()将顺序流转化为并行流
        Stream<Integer> parallelStream2 = list.stream().parallel();

        int[] arr = {1, 2, 3};
        // 通过数组创建流
        IntStream stream2 = Arrays.stream(arr);

        // 通过Stream的静态方法创建流
        Stream<Integer> stream3 = Stream.of(1, 2, 3);
        Stream<Integer> stream4 = Stream.iterate(1, x -> x + 1).limit(3);

        stream1.forEach(x -> System.out.print(x + " "));
        System.out.println(); // 1 2 3

        parallelStream1.forEach(x -> System.out.print(x + " "));
        System.out.println(); // 随机
    }
}
```

- stream是顺序流，由主线程按顺序对流执行操作。
- parallelStream是并行流，内部以多线程并行执行的方式对流进行操作，但前提流中的数据处理没有顺序要求。并行流能充分利用cpu优势，在数据量足够大的时候，加快处理速度。

## 测试API

### 新建测试数据

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class Book {

    Long id;
    String title;
    String author;
    Integer pageCount;
    Double price;
}

```

```java
public class TestStream {

    List<Book> bookList = new ArrayList<>();

    @Before
    public void init() {
        bookList.add(Book.builder().author("天乔巴夏").id(1L).title("Java-Spring").pageCount(100).price(50d).build());
        bookList.add(Book.builder().author("summerday").id(2L).title("Java-SpringBoot").pageCount(200).price(100d).build());
        bookList.add(Book.builder().author("hyh").id(3L).title("mysql").pageCount(500).price(150d).build());
        bookList.add(Book.builder().author("tqbx").id(4L).title("Linux").pageCount(30).price(10d).build());
    }
}
```

### findFirst、findAny

```java
        // 匹配第一个
        Optional<Book> first = bookList.stream().filter(book -> book.getPageCount() > 100).findFirst();
        first.ifPresent(book -> System.out.println("匹配第一个值 : " + book));

        // 匹配任意
        Optional<Book> any = bookList.parallelStream().filter(book -> book.getPageCount() > 100).findAny();
        any.ifPresent(book -> System.out.println("匹配任意的值 : " + book));
```

### anyMatch、noneMatch

```java
        // 是否包含符合条件的书
        boolean anyMatch = bookList.stream().anyMatch(book -> book.getPageCount() > 100);
        System.out.println("是否存在页数大于100的书 : " + anyMatch);

        // 检查是否有名字长度大于5 的
        boolean noneMatch = bookList.stream().noneMatch(book -> (book.getTitle().length() > 5));
        System.out.println("不存在title长度大于5的书 : " + noneMatch);
```

### filter

```java
        // 找到所有id为奇数的书,列出他们的书名到list中
        List<String> titleList = bookList.stream()
                .filter(book -> book.getId() % 2 == 1)
                .map(Book::getTitle)
                .collect(Collectors.toList());
        System.out.println(titleList);
```

### max、count

```java
        // 获取页数最多的书
        Optional<Book> max = bookList.stream().max(Comparator.comparingInt(Book::getPageCount));
        max.ifPresent(book -> System.out.println("页数最多的书 : " + book));
        // 计算mysql书籍有几本
        long count = bookList.stream().filter(book -> book.getTitle().contains("mysql")).count();
        System.out.println("mysql书籍的本数 : " + count);

```

### peek、map

```java
        // 将所有的书的价格调高100并输出调高以后的书单
        List<Book> result = bookList.stream().peek(book -> book.setPrice(book.getPrice() + 100))
                .collect(Collectors.toList());
        result.forEach(System.out::println);
        // 获取所有书的id列表
        List<Long> ids = bookList.stream().map(Book::getId).collect(Collectors.toList());
        System.out.println(ids);
```

### reduce

```java
        // 求所有书籍的页数之和
        Integer totalPageCount = bookList.stream().reduce(0, (s, book) -> s += book.getPageCount(), Integer::sum);
        System.out.println("所有书籍的页数之和 : " + totalPageCount);
```

###  collect

```java
        // 将所有书籍存入 author -> title 的map中
        Map<String, String> map = bookList.stream().collect(Collectors.toMap(Book::getAuthor, Book::getTitle));
        // 取出所有id为偶数的书,存入list
        List<Book> list = bookList.stream().filter(book -> book.getId() % 2 == 0).collect(Collectors.toList());
        // 取出所有标题长度大于5的书,存入list
        Set<Book> set = bookList.stream().filter(book -> book.getTitle().length() > 5).collect(Collectors.toSet());
```

### count、averaging、summarizing、max、sum

```java
        // 统计书籍总数
        Long bookCount = bookList.stream().filter(book -> "天乔巴夏".equals(book.getAuthor())).count();

        // 求平均价格
        Double average = bookList.stream().collect(Collectors.averagingDouble(Book::getPrice));

        // 求最贵价格
        Optional<Integer> max = bookList.stream().map(Book::getPageCount).max(Double::compare);

        // 求价格之和
        Integer priceCount = bookList.stream().mapToInt(Book::getPageCount).sum();

        // 一次性统计所有信息
        DoubleSummaryStatistics c = bookList.stream().collect(Collectors.summarizingDouble(Book::getPrice));
```

### group

```java
        // 按书的价格是否高于100分组
        Map<Boolean, List<Book>> part = bookList.stream().collect(Collectors.partitioningBy(book -> book.getPrice() > 100));

        for (Map.Entry<Boolean, List<Book>> entry : part.entrySet()) {
            if (entry.getKey().equals(Boolean.TRUE)) {
                System.out.println("price > 100 ==> " + entry.getValue());
            } else {
                System.out.println("price <= 100 <== " + entry.getValue());
            }
        }

        // 按页数分组
        Map<Integer, List<Book>> group = bookList.stream().collect(Collectors.groupingBy(Book::getPageCount));
        System.out.println(group);
```

### join

```java
        // 获取所有书名
        String titles = bookList.stream().map(Book::getTitle).collect(Collectors.joining(","));
        System.out.println("所有书名 : " + titles);
```

### sort

```java
        // 按价格升序
        List<Book> sortListByPrice = bookList.stream().sorted(Comparator.comparing(Book::getPrice)).collect(Collectors.toList());
        System.out.println(sortListByPrice);
        // 按价格降序
        List<Book> sortListByPriceReversed = bookList.stream().sorted(Comparator.comparing(Book::getPrice).reversed()).collect(Collectors.toList());
        System.out.println(sortListByPriceReversed);

        // 先价格再页数
        List<Book> sortListByPriceAndPageCount = bookList.stream().sorted(Comparator.comparing(Book::getPrice)
                .thenComparing(Book::getPageCount)).collect(Collectors.toList());
        System.out.println(sortListByPriceAndPageCount);
```

### distinct、concat、limit、skip

```java
        Stream<Integer> stream1 = Stream.of(1, 2, 2, 3, 4);
        Stream<Integer> stream2 = Stream.of(2, 3, 4, 5, 5);
        // 合并
        List<Integer> concatList = Stream.concat(stream1, stream2).collect(Collectors.toList());
        System.out.println(concatList);

        // 去重
        List<Integer> distinctList = concatList.stream().distinct().collect(Collectors.toList());
        System.out.println(distinctList);

        // 限制
        List<Integer> limitList = distinctList.stream().limit(3).collect(Collectors.toList());
        System.out.println(limitList);

        // 跳过
        List<Integer> skipList = limitList.stream().skip(1).collect(Collectors.toList());
        System.out.println(skipList);

        // 迭代
        List<Integer> iterateList = Stream.iterate(1, x -> x + 2).limit(10).collect(Collectors.toList());
        System.out.println(iterateList);

        // 生成
        List<Integer> generateList = Stream.generate(() -> new Random().nextInt()).limit(5).collect(Collectors.toList());
        System.out.println(generateList);
```

## 参考阅读

- [Java8 Stream：2万字20个实例，玩转集合的筛选、归约、分组、聚合](https://blog.csdn.net/mu_wind/article/details/109516995)