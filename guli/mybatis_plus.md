# Mybatis_plus入门

```xml
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.3.2</version>
        </dependency>
```

# 主键生成策略

- 自动增长AUTO INCREMENT，分表的时候下一个表需要获取上一表的最后值。
- uuid，每次生成唯一随机值，无法排序，无序关注上表。
- redis实现，incr和incrby两个原子操作实现。
- 雪花算法snowflake，mybatis_plus自带的算法。

https://www.cnblogs.com/haoxinyue/p/5208136.html

```java
@Getter
public enum IdType {
    /**
     * 数据库ID自增
     */
    AUTO(0),
    /**
     * 该类型为未设置主键类型(注解里等于跟随全局,全局里约等于 INPUT)
     */
    NONE(1),
    /**
     * 用户输入ID
     * <p>该类型可以通过自己注册自动填充插件进行填充</p>
     */
    INPUT(2),

    /* 以下3种类型、只有当插入对象ID 为空，才自动填充。 */
    /**
     * 分配ID (主键类型为number或string）,
     * 默认实现类 {@link com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator}(雪花算法)
     *
     * @since 3.3.0
     */
    ASSIGN_ID(3),
    /**
     * 分配UUID (主键类型为 string)
     * 默认实现类 {@link com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator}(UUID.replace("-",""))
     */
    ASSIGN_UUID(4),
    /**
     * @deprecated 3.3.0 please use {@link #ASSIGN_ID}
     */
    @Deprecated
    ID_WORKER(3),
    /**
     * @deprecated 3.3.0 please use {@link #ASSIGN_ID}
     */
    @Deprecated
    ID_WORKER_STR(3),
    /**
     * @deprecated 3.3.0 please use {@link #ASSIGN_UUID}
     */
    @Deprecated
    UUID(4);

    private final int key;

    IdType(int key) {
        this.key = key;
    }
}
```

# 自动填充

```java
    @TableField(fill = FieldFill.INSERT)
    private Date createTime;

    @TableField(fill =FieldFill.INSERT_UPDATE)
    private Date updateTime;
```

```java
/**
 * mybatis_plus自动填充配置
 *
 * @author Summerday
 */

@Component
public class MyMetaObjectHandler implements MetaObjectHandler {


    /**
     * 添加操作时,会执行这个方法
     *
     * @param metaObject
     */
    @Override
    public void insertFill(MetaObject metaObject) {
        this.setFieldValByName("createTime", new Date(), metaObject);
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }

    /**
     * 修改操作时,会执行这个方法
     *
     * @param metaObject
     */
    @Override
    public void updateFill(MetaObject metaObject) {
        this.setFieldValByName("updateTime", new Date(), metaObject);
    }
}
```

# 乐观锁

并发情况下，读操作的问题：脏读，不可重复读，幻读。

并发情况下，写操作出现的丢失更新问题，多个人同时修改统一条记录，最后提交将会把之前提交给覆盖，解决方案：

- 悲观锁，串行
- 乐观锁，version字段，在提交之前需要比较当前版本号是否和数据库中相同，相同就可以提交，并且将版本号加一。

```java
    @Version
    @TableField(fill = FieldFill.INSERT)
    private Integer version;
```

配置乐观锁插件

```java
    /**
     * 乐观锁插件
     * @return 乐观锁插件
     */
    @Bean
    public OptimisticLockerInterceptor optimisticLockerInterceptor(){
        return new OptimisticLockerInterceptor();
    }
```

```java
    @Test
    public void testOptimistic(){
        //根据id查询数据
        User user = userMapper.selectById(1286653405875503106L);

        //修改
        user.setAge(20);
        userMapper.updateById(user);

    }
```

最终sql的执行语句是：

`UPDATE user SET name=?, age=?, create_time=?, update_time=?, version=? WHERE id=? AND version=? `

# 分页

配置分页插件

```java
    @Bean
    public PaginationInterceptor paginationInterceptor(){
        return new PaginationInterceptor();
    }
```

```java
    @Test
    public void testPage(){
        //创建page对象,传入当前页和每页显示条数
        Page<User> page = new Page<>(1,5);

        //调用mp分页的查询方法,分页的所有数据在page对象，暂时条件为空
        userMapper.selectPage(page, null);

        //通过page对象获取数据
        System.out.println(page.getCurrent()); //当前页
        System.out.println(page.getRecords()); //每页数据list集合
        System.out.println(page.getSize()); //每页显示记录数
        System.out.println(page.getTotal()); //总记录数
        System.out.println(page.getOrders()); //排序字段信息
        System.out.println(page.getPages()); //总页数
        System.out.println(page.hasNext());  //是否有下页
        System.out.println(page.hasPrevious()); //是否有上页
    }
```

# 物理删除和逻辑删除

物理单个删除

```java
    @Test
    public void delete(){
        int i = userMapper.deleteById(1L);
        System.out.println(i);
    }
```

 物理批量删除

```java
    @Test
    public void deleteBatchIds(){
        int i = userMapper.deleteBatchIds(Arrays.asList(2, 3));
        System.out.println(i);
    }
```

逻辑删除

```java
    @TableLogic
    private Integer deleted;
```

3.3.0版本之后，如果进行以下配置，就不需要配置@TableLogic了

```properties
mybatis-plus.global-config.db-config.logic-delete-field=deleted
mybatis-plus.global-config.db-config.logic-delete-value=1
mybatis-plus.global-config.db-config.logic-not-delete-value=0
```

# 条件查询

```java
    @Test
    public void testWrapper(){

        //创建wrapper对象
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        //通过wrapper设置条件
        //ge >= gt > le <= lt<
        //eq  = ne !=
        //between in()
        //like 模糊
        //orderByDesc/Asc 排序
        //last 可以拼接在sql最后
        //select指定要查询的列
        wrapper.ge("age",20);//age>=20
        List<User> users = userMapper.selectList(wrapper);
        System.out.println(users);
    }
```

