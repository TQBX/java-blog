# 解决：Specified key was too long; max key length is 767 bytes

## 问题产生：

经过查找资料，应该是在给一个varchar(255)类型的字段建立索引的时候，超过了767字节的长度。

## 解决办法：

可以适当修改字段的长度，修改细节需要根据编码格式来考虑。

如果字段长度为100，编码方式为utf8，那么最大占用300字节，但如果是utf8mb4，则最大占用400字节。

另：数据库的编码格式可以通过以下命令查看：

```sql
SHOW VARIABLES LIKE 'character_set_database';
```

可以通过以下命令设置：

```sql
ALTER DATABASE `table_name` CHARACTER SET utf8 ;
```

