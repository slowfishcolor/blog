---
title: Spring 的事务管理
date: 2018/12/18 19:38:00
category: Spring
tags: 
- Java
- Spring
---

强大的事务管理是 Spring 的一大优点，本文将介绍 Spring 的事务核心抽象，在此基础上介绍 Spring 的声明式事务和编程式事务，并给出一定的实践建议。

## Spring 的事务抽象

Spring 事务的核心抽象是 `PlatformTransactionManager` 接口。

```java
public interface PlatformTransactionManager {

    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;

    void rollback(TransactionStatus status) throws TransactionException;
}
```

该接口主要被用作 SPI (Service Provider Interface)，不同的数据持久化提供方去实现该接口，比如 JPA 的`org.springframework.orm.jpa.JpaTransactionManager`。同时在编程式事务中可以直接使用该接口进行事务操作。

该接口方法抛的异常 `TransactionException` 是非受检异常，这非常符合 Spring 的设计哲学，非常简单，用户通常不用去处理，因为真的抛这个异常的时候往往就是非常非常严重的错误了。

`getTransaction(..)` 会根据传入的 `TransactionDefinition` 返回对应的 `TransactionStatus`。`TransactionStatus` 可能代表了一个新的事务，也可能代表了一个已经存在的事务。

`TransactionDefinition` 接口定义了如下事务属性：

* `Isolation`: 事务的隔离级别，即在一个事务中能看到并发执行的其他事务的程度，比如脏读、幻读、不可重复读等。
* `Propagation`: 事务的传播属性，即事务的作用域，比如是加入当前存在的事务，还是新起一个事务。
* `Timeout`: 事务的超时时间。
* `Read-only status`: 是否是只读事务，这个主要是优化用的，比如 `Hibernate` 会依赖这个属性进行优化。

`TransactionStatus` 用来记录事务的状态，提供了控制事务执行和查询事务状态的简单接口。

## 声明式事务管理

声明式事务代码侵入性小，不用写样板代码，因此是十分理想的事物管理方式。

声明式事务是基于 Spring 的 AOP (Aspect-Oriented Programming) 来实现的。大体原理如下图所示：

![声明式事务原理](tx.png)

## @Transactional 注解

Spring 与 JTA (Java Transaction API) 都有这个注解，分别是 `org.springframework.transaction.annotation.Transactional` 和 `javax.transaction.Transactional`。Spring 在 4.0 版本之后[正式支持 JTA 的注解](https://jira.spring.io/browse/SPR-9139)。之前版本(2.5.3 - 3.2.5) 只有在 EJB3 环境下才支持，即配合 J2EE 使用时才支持。与 JTA 的 Transaction 注解相比，Spring 的注解提供了更多配置项，如事务隔离级别、超时时间、read-only 等。在 Spring 项目中还是推荐使用 Spring 的 @Transactional 注解。

## Reference

* [Spring Framework Reference, Transaction Management](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/transaction.html)
* [Transactions, Caching and AOP: understanding proxy usage in Spring](https://spring.io/blog/2012/05/23/transactions-caching-and-aop-understanding-proxy-usage-in-spring)