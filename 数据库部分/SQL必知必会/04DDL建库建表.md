## 基础语法

> DDL 是 DBMS 的核心组件，也是 SQL 的重要组成部分，DDL 的正确性和稳定性是整个 SQL 运行的重要基础。

Data Detinition Language，数据定义语言，定义了数据库的结构和数据表的结构。

常用增删改功能： create , drop, alter，在执行DDL的时候，不需要commit，就可以完成执行任务。

**定义数据库**

```sql
create database xx;
drop database xx;
```

**定义数据表**

```sql
CREATE TABLE player ( 
    player_id int(11) NOT NULL AUTO_INCREMENT, 
 	player_name varchar(255) NOT NULL
);
```

- int(11) 代表整数类型，显示长度为11位，11是最大有效显示长度，与类型包含的数据范围大小无关。
- varchar(255) 代表最大长度为255的可变字符串类型。
- not null 表明该字段不能为null。
- auto_increment代表主键自动增长。

**修改表结构**

添加age字段，类型为int(11)

```sql
ALTER TABLE player ADD (age int(11));
```

修改字段名

```sql
ALTER TABLE player RENAME COLUMN age to player_age
```

修改字段的数据类型

```sql
ALTER TABLE player MODIFY COLUMN player_age float(3,1);
```

删除字段

```sql
ALTER TABLE player DROP COLUMN player_age;
```

## 常见约束

**主键约束**

- 唯一标识一条记录，不能重复，不能为空，可以理解为`unique + not null`。

- 一个数据表的主键只能有一个。

- 主键可以是一个字段，也可以由多个字段复合组成。

**外键约束**

- 保证表与表之间引用的完整性。

**唯一性约束**

- 表明字段在表中的数值是唯一的。

**非空约束**

- 对字段定义not null，表明字段不能为空，必须有值。

**default**

- 表明字段的默认值，插入时，如果字段没有取值，就设置为默认值。

**check**

- 用来检查特定字段取值范围的有效性，check约束结果不能为false。