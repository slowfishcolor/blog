---
title: Spring 的事务管理
date: 2018/12/18 19:38:00
category: Spring
tags: 
- Java
- Spring
- Transaction
---

便捷的事务管理是 Spring 的一大优点，本文将介绍 Spring 的事务核心抽象，在此基础上介绍 Spring 的编程式事务和声明式事务，并给出一定的实践建议。

<!-- more -->

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

## 编程式事务管理

Spring 的编程式事务就是在代码中直接操作 `PlatformTransactionManager`，来配置、开启、提交、回滚。一个简单的示例如下：

```java
@Autowired
private PlatformTransactionManager transactionManager;

public void transfer(Integer amount, String userId) {
    DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
    // explicitly setting the transaction name only  can be done programmatically
    definition.setName("SomeTxName");
    definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

    TransactionStatus status = transactionManager.getTransaction(definition);
    try {        
        // execute your business logic here
        modify(userId, amount);

        transactionManager.commit(status);
    } catch (Exception e) {
        transactionManager.rollback(status);
            throw e;
        }
    }
```

当然，Spring 也提供了 `TransactionTemplate`，使用者只要向 `execute(..)` 方法传递包含业务代码的 `TransactionCallback` 即可将业务代码运行在事务环境中。

## 声明式事务管理

声明式事务即用户通过 xml 或注解的方式来声明需要开启事务的业务代码，Spring 自动为其进行事务管理。与编程式事务相比，声明式事务代码侵入性小，不用写样板代码，因此是十分理想的事务管理方式。

声明式事务是基于 Spring 的 AOP (Aspect-Oriented Programming) 来实现的。基本原理如下图所示：

![声明式事务原理](tx.png)

在 bean 的加载阶段，当看到 bean 中包含了需要开启事务的方法，Spring 便会为其生成代理对象，其他 bean 在注入该 bean 的依赖时，注入的是代理对象。这样，其他 bean 只能通过代理对象来调用目标 bean 的方法。当调用需要开启事务的方法时，代理对象首先调用事务增强 `TransactionInterceptor`，事务增强根据事务配置和当前事务状态，决定是开启新的事务还是沿用当前事务，之后调用 AOP 的其他增强，最总才调用到用户编写的业务代码。事务切面是一个 `around` 切面，在之行完用户的业务代码和其他切面的增强后，事务增强根据代码执行结果决定是 `commit` 还是 `rollback` 当前事务。 

## @Transactional 注解

当使用声明式事务管理时，可以使用 `@Transactional` 注解进行事务的声明。在声明事务时，用户可以配置 `@Transactional` 的 `propagation` 事务传播属性、`rollbackFor` 回滚的异常等。

其中，`rollbackFor` 的默认配置是只回滚 `RuntimeException` 和 `Error`，也就是说 **默认配置下所有的受检异常都不会回滚**。因此，如果在抛出受检异常时需要回滚，用户需要显示的配置 `rollbackFor = CheckedException.class`。某些特殊情况下，如果必须要截断异常，不能上抛，可在 `catch` 代码块中通过 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();` 来显示的设定 `rollbackOnly`，保证事务方法的回滚。

值得注意的是，Spring 与 JTA (Java Transaction API) 都有这个注解，分别是 `org.springframework.transaction.annotation.Transactional` 和 `javax.transaction.Transactional`。Spring 在 4.0 版本之后 [正式支持 JTA 的注解](https://jira.spring.io/browse/SPR-9139)。之前版本 (2.5.3 - 3.2.5) 只有在 EJB3 环境下才支持，即配合 J2EE 使用时才支持。与 JTA 的 Transaction 注解相比，Spring 的注解提供了更多配置项，如事务隔离级别、超时时间、read-only 等。在 Spring 项目中还是推荐使用 Spring 的 @Transactional 注解。


## 声明事务的有效性 —— 理解代理的局限

Spring AOP 生成的动态代理对象需要继承目标类并覆写需要代理的方法，因此 **代理方法必须是可以被覆写的**，即不是 final 方法或 private 方法。

同时，只有通过代理对象调用的事务方法才会进入到事务切面，因此，直接通过 this 调用的方法无法有效的开启事务。如下是一个典型的事务声明无效的场景：

```java
@Service
public class SimplePojo implements Pojo { 

    public void foo() { 
        // this next method invocation is a direct call on the 'this' reference 
        this.bar(); 
    }
 
    @Transactional
    public void bar() { 
        // some logic... 
    } 
} 
```

其他 bean 直接调用 `bar()` 时事务是有效的，但当调用 `foo()` 方法时，由于该方法没有声明需要事务，因此代理对象不会去执行事务增强来开启事务，而 `foo()` 方法在调用 `bar()` 方法时是通过 `this` 直接引用的，并不会经过代理对象，因此这种情况下 `bar()` 的事务声明无效，不会开启事务。

![无效的事务](proxy.png)


## Reference

* [Spring Framework Reference, Transaction Management](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/transaction.html)
* [Transactions, Caching and AOP: understanding proxy usage in Spring](https://spring.io/blog/2012/05/23/transactions-caching-and-aop-understanding-proxy-usage-in-spring)