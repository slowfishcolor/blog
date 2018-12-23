---
title: JPA / Hibernate 的逻辑删除
date: 2018/9/19 11:39:00
category: ORM
tags: 
- JPA
- Hibernate
- Spring Data
---

JPA 为数据库操作提供了极大的方便，但很多场景下仍不够灵活。Hibernate 作为 JPA 标准的实现提供方之一，提供了一定的拓展，丰富了 JPA 的功能。本文就实际项目中常见的逻辑删除问题，给出了 Hibernate 的解决方案。

<!-- more -->

## 逻辑删除定义

逻辑删除是指在删除数据库的某条记录时，并不是真正的将该条记录删除，而是通过某个字段来标识其状态为“删除”，在接下来的查询等操作时，根据此字段来过滤调被删除的记录。

## 使用 Hibernate 进行逻辑删除

进行逻辑删除时，需要覆盖 Hibernate 默认的删除操作，告诉 Hibernate 使用特定的 UPDATE 语句来代替 DELETE。这一操作可以使用 `@SQLDelete` 注解来进行。

使用者需要为 `@SQLDelete` 注解给定 `sql` 字段。Hibernate 在执行删除操作，生成原生 SQL 时，会使用该字段。因此，该字段需要是数据库原生的 SQL。之后，不论是使用 Spring Data Jpa 提供的  SimpleJpaRepository 还是使用 `@Query` 注解来写 `JPQL`来执行删除操作，Hibernate 都会使用用户指定的 sql 来代替默认的 delete 语句。当然，用户写的原生 SQL 是不会进行该替换的。

## 更新当前 session 中的删除状态

由于 `@SQLDelete` 的原理是在生成 SQL 语句的时候对默认的 delete 语句进行替换，因此，Hibernate 并不知道他所管理的 Entity 的相应字段在执行过删除操作后进行了改变。通常情况下，这并没有什么问题，因为只要 Hibernate 执行了逻辑删除的 SQL，新的实体在查询的时候就会感知到删除状态。然而，在某些情况下，如执行了 `EntityManager.remove(Object entity)` 后，并没有立即 flush 到数据库，并立即对该 entity 进行操作，那么此时 entity 的相应字段并不会变为删除状态。解决这种情况的方法就是通过 `@PreRemove` 注解来注册一个回调函数，在该函数中更新相应字段的状态。这样，Hibernate 在执行删除之前，会回调该方法，从而更新相应字段的状态。

## 查询时避开逻辑删除的记录

执行逻辑删除后，通常不希望在执行查询操作时查询到逻辑删除过的记录，Hibernate 给出的解决方案是通过 `@Where` 注解。`@Where` 注解类似于 `@SQLDelete` 注解，用户在这里提供自定义的原生 where 语句，Hibernate 在生成真正的 SQL where 语句时，会将该语句与目标语句的查询条件通过 and 进行连接。用户可以将未逻辑删除的限定条件通过该注解进行设定，之后，不论是 `JPQL` 还是 `EntityManager` 相关的查询操作，都会带上该条件。

## 示例

本部分给出一个简单的示例，如下所示。User 的 deleted 域标明了该实体的删除状态。通过 `@SQLDelete` 注解指定删除操作时要执行的 update 操作，通过 `@Where` 注解指定了查询时的拼接条件。通过 `@PreRemove` 注解指定执行删除操作时的状态变更操作。

```java
import lombok.Data;
import org.hibernate.annotations.SQLDelete;
import org.hibernate.annotations.Where;
import javax.persistence.*;

@Data
@Entity
@SQLDelete(sql = "update user set deleted = 1 where id = ?")
@Where(clause = "deleted = 0")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    /**
     * 逻辑删除标志位，0 未删除，1 已删除
     */
    private Integer deleted;

    @PreRemove
    public void deleteUser() {
        this.deleted = 1;
    }
}
```

在执行删除时，Hibernate 生成的语句如下：

```
Hibernate: update user set deleted = 1 where id = ?
```

执行查询操作时，Hibernate 生成语句如下：

```
Hibernate: select user0_.id as id1_0_0_, user0_.deleted as deleted2_0_0_, user0_.name as name3_0_0_ from user user0_ where user0_.id=? and ( user0_.deleted = 0)
```

## 总结

借助 Hibernate 提供的 `@SQLDelete` 注解、`@Where` 注解，可以方便的实现 JPA 的逻辑删除。配合 Spring Data JPA 等优秀的开源项目，我们可以更加灵活方便的使用 JPA。

## Reference

* [How to implement a soft delete with Hibernate](https://www.thoughts-on-java.org/implement-soft-delete-hibernate/)