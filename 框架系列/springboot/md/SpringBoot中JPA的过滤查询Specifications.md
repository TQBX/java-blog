SpringData JPA遵循Eric Evans在`Domain Driver Design`一书中的规范，让你可以使用编程方式来构建多条件查询。

## 快速开始

关于SpringBoot与JPA的快速整合，已经在这篇文章中写的非常详细：[SpringBoot整合Spring Data JPA](https://www.cnblogs.com/summerday152/p/14054637.html)，一些配置部分就不再赘述了，我们直接创建一个条件丰富一些的实体类做测试。

### 创建实体类

```java
@Entity(name = "t_blog")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Blog implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /**
     * 文章名称
     */
    private String name;
    /**
     * 作者
     */
    private String author;

    /**
     * 状态, 0代表未发布, 1代表已发布
     */
    private Integer status;

    /**
     * 发布时间
     */
    @Column(name = "publish_time")
    private Date publishTime;

    /**
     * 更新时间
     */
    @Column(name = "update_time")
    private Date updateTime;

}
```

### 继承JpaSpecificationExecutor接口

```java
public interface BlogDao extends JpaRepository<Blog, Long>, JpaSpecificationExecutor<Blog> {
}
```

继承`JpaSpecificationExecutor`之后，便能够使用其中提供`Specification`相关的方法：

```java
public interface JpaSpecificationExecutor<T> {
	Optional<T> findOne(@Nullable Specification<T> spec);
	List<T> findAll(@Nullable Specification<T> spec);
	Page<T> findAll(@Nullable Specification<T> spec, Pageable pageable);
	List<T> findAll(@Nullable Specification<T> spec, Sort sort);
	long count(@Nullable Specification<T> spec);
}
```

### Specification接口

可以看到，他们都需要一个Specification接口对象，而且可以注意到这个接口中有Java8新增的接口规范，如static方法、default方法，以及仅有一个抽象方法，可以作为函数式接口，传入Lambda表达式简化书写。

这些方法的用法其实见名知义对吧。

```java
public interface Specification<T> extends Serializable {

	long serialVersionUID = 1L;

	static <T> Specification<T> not(@Nullable Specification<T> spec) {

		return spec == null //
				? (root, query, builder) -> null //
				: (root, query, builder) -> builder.not(spec.toPredicate(root, query, builder));
	}

	static <T> Specification<T> where(@Nullable Specification<T> spec) {
		return spec == null ? (root, query, builder) -> null : spec;
	}

	default Specification<T> and(@Nullable Specification<T> other) {
		return SpecificationComposition.composed(this, other, CriteriaBuilder::and);
	}

	default Specification<T> or(@Nullable Specification<T> other) {
		return SpecificationComposition.composed(this, other, CriteriaBuilder::or);
	}

	@Nullable
	Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);
}
```

### 复杂查询

```java
    @Test
    public void findByExample() {
        Specification<Blog> specification = new Specification<Blog>() {
            @Override
            public Predicate toPredicate(Root<Blog> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
                List<Predicate> predicates = new LinkedList<>();
                predicates.add(cb.equal(root.get("status"), 1)); // 查询status状态为1
                predicates.add(cb.like(root.get("author"), "%summer%")); // author名称中存在summer
                predicates.add(cb.between(root.get("publishTime"),  // 发布时间7天前至今
                        Date.from(Instant.now().minus(Duration.ofDays(7))), new Date()));

                return query.where(predicates.toArray(new Predicate[0])).getRestriction();
            }
        };
        // 按照条件查询,并按照publish_time升序
        List<Blog> blogs
                = blogDao.findAll(specification, Sort.by(Sort.Direction.ASC, "publishTime"));
        blogs.forEach(System.out::println);
    }
```

最后的查询语句如下：

```sql
select id, author, name,publish_time,status,update_time from t_blog
where status = 1 and (author like ?)
and (publish_time between ? and ?)
order by publish_time asc
```

## 参考阅读

- https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference