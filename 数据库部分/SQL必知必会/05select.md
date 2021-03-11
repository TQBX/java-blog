## 基础语法

**查询列**

```sql
select name from heros;
```

**查询多个列**

```sql
select name, hp_max from heros;
```

**查询所有列**

```sql
select * from heros;
```

**起别名**

```sql
select name as n, hp_max as hm from heros;
```

**查询常数**

```sql
select '王者荣耀' as platform, name from heros;
```

![img](img/Untitled/41ed73cef49e445d64b8cb748a82c299.png)

**去除重复行**

```sql
select distinct attack_range from heros;
```

![img](img/Untitled/e67c0d2f7b977cb0ff87891eb9adf615.png)

```sql
select distinct attack_range, name from heros;
```

![img](img/Untitled/0105eb3f0b74d0ed5e6c2fafca38292a.png)

- DISTINCT 需要放在所有列名前面。
- DISTINCT 其实是对后面**所有列名的组合**进行去重。

**排序检索**

1. 排序的列名：ORDER BY 后面可以有一个或多个列名，**如果是多个列名进行排序，会按照后面第一个列先进行排序，当第一列的值相同的时候，再按照第二列进行排序**，以此类推。
2. 排序的顺序：ORDER BY 后面可以注明排序规则，**ASC 代表递增排序，DESC 代表递减排序**。如果没有注明排序规则，默认情况下是按照 **ASC** 递增排序。我们很容易理解 ORDER BY 对数值类型字段的排序规则，但如果排序字段类型为文本数据，就需要参考数据库的设置方式了，这样才能判断 A 是在 B 之前，还是在 B 之后。比如使用 MySQL 在创建字段的时候设置为 BINARY 属性，就代表区分大小写。
3. 非选择列排序：ORDER BY 可以使用非选择列进行排序，所以即使在 SELECT 后面没有这个列名，你同样可以放到 ORDER BY 后面进行排序。
4. ORDER BY 的位置：**ORDER BY 通常位于 SELECT 语句的最后一条子句**，否则会报错。

```sql
SELECT name, hp_max FROM heros ORDER BY hp_max DESC -- 最大生命值从高到低
SELECT name, hp_max FROM heros ORDER BY mp_max, hp_max DESC -- 多列排序
```

**约束返回结果的数量**

```sql
SELECT name, hp_max FROM heros ORDER BY hp_max DESC LIMIT 5
```

约束返回结果的数量可以减少数据表的网络传输量，也可以提升查询效率。如果我们知道返回结果只有 1 条，就可以使用LIMIT 1，告诉 SELECT 语句只需要返回一条记录即可。这样的好处就是 SELECT 不需要扫描完整的表，只需要检索到一条符合条件的记录即可返回。

## select 的执行顺序

1. 关键字的顺序不能颠倒

```sql
SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ...
```

2. select语句的执行顺序

```sql
FROM > WHERE > GROUP BY > HAVING > SELECT的字段 > DISTINCT > ORDER BY > LIMIT
```

```sql
SELECT DISTINCT player_id, player_name, count(*) as num #顺序5
FROM player JOIN team ON player.team_id = team.team_id #顺序1
WHERE height > 1.80 #顺序2
GROUP BY player.team_id #顺序3
HAVING num > 2 #顺序4
ORDER BY num DESC #顺序6
LIMIT 2 #顺序7
```

