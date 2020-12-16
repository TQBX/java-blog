## Clock

```java
    @Test
    public void clock() {
        // 获取当前clock
        Clock clock = Clock.systemUTC();
        // 获取当前时刻
        Instant instant = clock.instant();
        System.out.println(instant);
        // 获取对应的毫秒数
        System.out.println(clock.millis());
        System.out.println(System.currentTimeMillis());

    }
```

## Duration

```JAVA
    @Test
    public void duration() {
        Clock now = Clock.systemUTC();
        System.out.println(now.instant());

        // 转换
        Duration duration = Duration.ofSeconds(3600 * 24);
        // 等价于多少分钟
        System.out.println(duration.toMinutes());
        // 等价于多少小时
        System.out.println(duration.toHours());
        // 等价于多少天
        System.out.println(duration.toDays());

        // 在 now 的基础之上加上1天
        Clock c = Clock.offset(Clock.systemUTC(), duration);
        System.out.println(c.instant());
    }
```

## Instant

```java
    @Test
    public void instant() {
        // 当前时刻  2020-12-13T10:45:39.478Z
        Instant now = Instant.now();
        System.out.println(now);

        // 一个小时之后  2020-12-13T11:45:39.478Z
        Instant instant = now.plusSeconds(3600);
        System.out.println(instant);

        // 根据字符串解析  2020-12-13T11:43:38.206Z
        Instant parse = Instant.parse("2020-12-13T11:43:38.206Z");
        System.out.println(parse);

        // 1天5小时之后 2020-12-14T15:47:22.886Z
        Instant plus = now.plus(Duration.ofDays(1).plusHours(5));
        System.out.println(plus);

        // 两天前 2020-12-11T10:47:22.886Z
        Instant minus = now.minus(Duration.ofDays(2));
        System.out.println(minus);
    }
```

## LocalDate

```java
    @Test
    public void localDate() {
        // 当前时间 2020-12-13
        LocalDate now = LocalDate.now();
        System.out.println(now);

        // 2020的第一天 2020-01-01
        LocalDate localDate = LocalDate.ofYearDay(2020, 1);
        System.out.println(localDate);

        // 1999年5月23号 1999-05-23
        LocalDate birth = LocalDate.of(1999, Month.MAY, 23);
        System.out.println(birth);
    }
```

## LocalTime

```java
    @Test
    public void localTime() {
        // 获取当前时间  18:56:20.245
        LocalTime now = LocalTime.now();
        System.out.println(now);
        // 设置为13:14
        LocalTime love = LocalTime.of(13, 14, 0);
        System.out.println(love);
        // 一天中的第一秒 00:00:01
        LocalTime localTime = LocalTime.ofSecondOfDay(1);
        System.out.println(localTime);
    }
```

## LocalDateTime

```java
    @Test
    public void localDateTime() {
        // 当前时间
        LocalDateTime now = LocalDateTime.now();
        System.out.println(now);

        // 1999-5-23 15:30
        LocalDateTime of = LocalDateTime.of(1999, 5, 23, 15, 30);
        System.out.println(of);
    }
```

## Year

```java
    @Test
    public void year(){
        // 获取当前的年份 2020
        Year now = Year.now();
        System.out.println(now);

        // 五年之后的年份 2025
        Year year = now.plusYears(5);
        System.out.println(year);
    }
```

## YearMonth

```java
    @Test
    public void yearMonth(){
        // 当前的2020-12
        YearMonth now = YearMonth.now();
        System.out.println(now);
        // 2020-05
        YearMonth yearMonth = YearMonth.of(Year.now().getValue(), 5);
        System.out.println(yearMonth);
        // 2025-10
        YearMonth ym = yearMonth.plusMonths(5).plusYears(5);
        System.out.println(ym);
    }
```

## MonthDay

```java
    @Test
    public void monthDay(){
        // 输出当前  --12-13
        MonthDay now = MonthDay.now();
        System.out.println(now);

        // 是这个月的第几天 13
        int dayOfMonth = now.getDayOfMonth();
        System.out.println(dayOfMonth);

        // 设置为 —-05-23
        MonthDay of = MonthDay.of(Month.MAY, 23);
        System.out.println(of);
    }
```

