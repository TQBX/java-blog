## SQL函数

SQL中的函数一般在数据上之心给，可以方便地转换和处理数据。

常用地SQL函数有哪些？

1. 算数函数
2. 字符串函数
3. 日期函数
4. 转换函数

## 算数函数

![img](img/07sql函数/193b171970c90394576d3812a46dd8e1.png)

## 字符串函数

![img](img/07sql函数/c161033ebeeaa8eb2436742f0f818a4d.png)

## 日期函数

![img](img/07sql函数/3dec8d799b1363d38df34ed3fdd29045.png)

```sql
SELECT extract(year FROM '2021-03-13'); -- 2021

SELECT current_date(); -- 2021-03-13
SELECT current_time(); --  14:10:09

SELECT current_timestamp(); -- 2021-03-13 14:10:09

SELECT date('2020-05-23 12:00:23'); -- 2020-05-23
SELECT time('2020-05-23 12:00:23'); -- 12:00:23
```

DATE 日期格式必须是 yyyy-MM-dd 的形式。

补充： mysql中str_to_date函数

```sql
SELECT str_to_date('03/13/2021','%m/%d/%Y'); -- 2021-03-13

SELECT str_to_date('20210313124304','%Y%m%d%H%i%s'); -- 2021-03-13 12:43:04

SELECT str_to_date('2021-03-13 12:43:04','%Y-%m-%d %H:%i:%s'); -- 2021-03-13 12:43:04
```

## 转换函数

![img](img/07sql函数/5d977d747ed1fddca3acaab33d29f459.png)

cast函数转换数据类型时，不会四舍五入，因此小数转整型时会报错。

小数可以转化为指定小数格式，mysql和sql server中可以使用`decimal(a,b)`指定，a表示整数部分+小数部分的最大位数，b表示小数位数。

```sql
SELECT cast(123.123 AS DECIMAL (8,2)); -- 123.12

-- select cast(1,2 as int); -- error

SELECT COALESCE (NULL,1,2); -- 1
```

## 案例

显示英雄以及他的物攻成长，对应字段为attack_growth，让这个字段精确到小数点后一位。

```sql
SELECT name, round(attack_growth, 1) FROM heros;
```

显示英雄最大生命值的最大值，就需要用到 MAX 函数。

```sql
SELECT max(hp_max) FROM heros;
```

最大生命值最大的是哪个英雄，以及对应的数值。

```sql
SELECT 
	name,
	hp_max 
FROM
	heros 
WHERE
	hp_max = ( SELECT max( hp_max ) FROM heros );
```

显示英雄的名字，以及他们的名字字数。

```sql
SELECT name, char_length(name) FROM heros;
```

提取英雄上线日期（对应字段 birthdate）的年份，只显示有上线日期的英雄即可（有些英雄没有上线日期的数据，不需要显示）

```sql
SELECT name, extract(year FROM birthdate) 
FROM heros 
WHERE birthdate IS NOT NULL;
```

```sql
SELECT name, year(birthdate) 
FROM heros 
WHERE birthdate IS NOT NULL;
```

找出在 2016 年 10 月 1 日之后上线的所有英雄。

```sql
SELECT 
	name,
	birthdate 
FROM
	heros 
WHERE
	birthdate > date( '2016-10-01' );
```

在 2016 年 10 月 1 日之后上线英雄的平均最大生命值、平均最大法力和最高物攻最大值。

```sql
SELECT
	avg( hp_max ),
	avg( mp_max ),
	max( attack_max ) 
FROM
	heros 
WHERE
	birthdate > date( '2016-10-01' );
```

