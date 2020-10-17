# MySQL中的Explain

```mysql
explain select 1;
```

![](img/exp.png)

| 列名          | 描述                                                   |
| ------------- | ------------------------------------------------------ |
| id            | 在一个大的查询语句中每个select关键字都对应一个唯一的id |
| select_type   | select关键字对应的那个查询的类型                       |
| table         | 表名                                                   |
| type          | 针对单表的访问方法                                     |
| possible_keys | 可能用到的索引                                         |
| key_len       | 实际使用到的索引长度                                   |
| ref           | 当使用索引列等值查询时，与索引列进行等值匹配的对象信息 |
| rows          | 预估的需要读取的记录条数                               |
| Extra         | 一些额外的信息                                         |

# 如何查看一个sql语句是否使用了索引

explain命令：

- extra出现Using filesort和Using temporary表示无法使用索引，需尽快优化。
- type出现index和all，表示走的是全表扫描，效率低下，这时需要对sql进行调优。
- 当type出现ref或者index时，表示走的是索引，index是标准不重复的索引，ref表示虽然使用了索引，但是索引列中有重复的值，但是就算有重复值，也只是在重复值的范围内小范围扫描，不造成重大的性能影响。



# 索引的类型

