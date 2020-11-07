## 问题描述

当查询语句中字段名为关键字时，会出现语法问题：

```sql
select group from user where id = 1;
```

## 解决方法

在字段两边加上`，【键盘中左上角Esc键下面那个】。修改语句为：

```sql
select `group` from user where id = 1;
```

MybatisPlus中的做法，在实体类字段上加上TableField注解：

```java
    @TableField("`group`")
    private String group;
```

