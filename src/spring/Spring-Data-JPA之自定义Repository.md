Spring-Data-JPA 提供许多常用的 Repository 接口，如 CrudRepository、PagingAndSortingRepository、JpaRepository 等。在实际开发中我们常常会有一些常用的自定义方法，那我们应该如何扩展 Repository 接口呢？

## 基类

首先定义一个 BaseRepository 继承 JpaRepository，并写下我们要扩展的方法

```java
@NoRepositoryBean
public interface BaseRepository<T, ID> extends JpaRepository<T, ID> {

    List<T> findByIdIn(Iterable<ID> ids);

    void deleteByIdIn(Iterable<ID> ids);
}
```

@NoRepositoryBean 的作用是通知 Spring 容器不要实例化 BaseRepository，因为 BaseRepository 是作为一个中间接口来派生具体的 Repository 接口。

接下来创建 BaseRepository 的实现类，实现扩展的方法

```java
public class BaseRepositoryImpl<T, ID> extends SimpleJpaRepository<T, ID> implements BaseRepository<T, ID> {

    private final EntityManager em;
    private final JpaEntityInformation<T, ?> entityInformation;

    public BaseRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {
        super(entityInformation, entityManager);
        this.em = entityManager;
        this.entityInformation = entityInformation;
    }

    @Override
    public List<T> findByIdIn(Iterable<ID> ids) {
        return findAllById(ids);
    }

    @Override
    public void deleteById(ID id) {
        Assert.notNull(id, "Id must not be null!");

        CriteriaBuilder builder = em.getCriteriaBuilder();
        CriteriaDelete<T> delete = builder.createCriteriaDelete(getDomainClass());
        Root<T> root = delete.from(getDomainClass());
        delete.where(builder.equal(root.get(entityInformation.getIdAttribute()), id));
        em.createQuery(delete).executeUpdate();
    }

    @Transactional
    @Override
    public void deleteByIdIn(Iterable<ID> ids) {
        Assert.notNull(ids, "Ids must not be null!");

        if (ids.iterator().hasNext()) {
            CriteriaBuilder builder = em.getCriteriaBuilder();
            CriteriaDelete<T> delete = builder.createCriteriaDelete(getDomainClass());
            Root<T> root = delete.from(getDomainClass());
            delete.where(getIdsPredicate(builder, root, ids));
            em.createQuery(delete).executeUpdate();
        }
    }

    @SuppressWarnings("unchecked")
    protected Predicate getIdsPredicate(CriteriaBuilder builder, Root<T> root, Iterable<ID> ids) {
        Path<ID> path = (Path<ID>) root.get(entityInformation.getIdAttribute());
        CriteriaBuilder.In<ID> inPredicate = builder.in(path);
        ids.forEach(inPredicate::value);
        return inPredicate;
    }
}
```
## 配置

最后我们可以通过 `@EnableJpaRepositories` 在主类上指定新的 repositoryBaseClass

```java
@EnableJpaRepositories(repositoryBaseClass = BaseRepositoryImpl.class)
@SpringBootApplication
public class JpaExamplesApplication {

    public static void main(String[] args) {
        SpringApplication.run(JpaExamplesApplication.class, args);
    }
}
```

