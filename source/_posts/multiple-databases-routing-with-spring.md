---
title: Spring 的多数据源方案
date: 2019/4/14 12:58:00
category: Spring
tags: 
- Java
- Spring
- DataSource
---

实际项目中，可能出现一个项目需要连接多个数据库的场景，针对该场景，本文给出了 Spring 框架中基于 `AbstractRoutingDataSource` 的多数据源解决方案。本文只给出比较简单的多数据源方案，不涉及复杂的分布式事务场景。

<!-- more -->

## 几种多数据源方案简介

Spring 项目中，对于多数据源的切换，有几种主流方案：

1. 配置多个上层的数据库操作对象，如 MyBaits 的 `SqlSessionFactory`、JPA 的 `EntityManager`、Spring 的 `JDBCTemplate`，为其配置不同的 `DataSource`，在使用的时候，按需注入，进行操作。
1. 配置多个 `PlatformTransactionManager`，分别配置不同的 `DataSource`，配合 `@Transactional` 的 `value` 属性来指定事务代码使用的具体 `PlatformTransactionManager`。
1. 使用 `AbstractRoutingDataSource` 来代理多个具体的 `DataSource`，在 `getConnection()` 时根据条件决定返回的具体数据源。

方案 1 由于需要指定操作时使用的数据库操作对象，使用起来侵入性较强，但是非常灵活。方案 2 可以直接使用声明式配置，侵入性很低，但是相关代码必须开启事务，灵活度较低，[Eugen Paraschiv 结合 JPA 给出了该方案的示例](https://www.baeldung.com/spring-data-jpa-multiple-databases)。方案 3 在 DataSource 级别直接进行切换，对上层对象没有要求，使用起来比较灵活，侵入性小，但需要开发者自己实现 `AbstractRoutingDataSource`。本文接下来就给出该方案的一个实现示例。

## 实现 `AbstractRoutingDataSource`

`AbstractRoutingDataSource` 代理多个具体的 `DataSource`，开发者需要覆写 `resolveSpecifiedDataSource` 方法，来指定返回的具体数据源。当调用 `getConnection()` 方法时，`AbstractRoutingDataSource` 会调用 `resolveSpecifiedDataSource` 来决定从哪个代理的 `DataSource` 中获取 `connection`。一个简单的实现如下：

```java
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContext.getCurrentDataSource();
    }
}
```

这里的 `DataSourceContext` 使用 `ThreadLocal` 变量来保存当前的数据源标识，这样就可以将操作使用的数据源与当前线程绑定，其简单实现示例如下：

```java
public class DataSourceContext {

    private static final ThreadLocal<DataSourceEnum> contextHolder = new InheritableThreadLocal<>();

    public static void use(DataSourceEnum dataSource) {
        contextHolder.set(dataSource);
    }

    public static DataSourceEnum getCurrentDataSource() {
        return contextHolder.get();
    }
}
```

在正确配置了 `RoutingDataSource` 代理的多个数据源后，开发者就可以在进行数据库操作前通过 `DataSourceContext` 指定要使用的具体数据源，不过这样会产生非常繁杂的样板代码，因此我们希望通过 AOP 来进行声明式的数据源切换。

## 使用 AOP 实现声明式数据源切换

AOP 切换的基本思路是使用注解进行数据源的声明，在切面的增强中解析注解，切换具体的数据源。首先进行注解的声明，示例如下：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface UseDataSource {

    DataSourceEnum value();
}
```

接下来声明下增强 `Advice`，在 `Advice` 中根据方法或类上声明的 `UseDataSource` 注解来切换 `DataSourceContext`，并在目标方法执行完毕后的 finally 代码块中切换回之前数据源。

```java
public class DataSourceRoutingAdvice {

    public Object aroundRoutingDataSource(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        if (method.getDeclaringClass().isInterface()) {
            method = joinPoint.getTarget().getClass().getMethod(methodSignature.getName(), methodSignature.getParameterTypes());
        }

        UseDataSource methodDataSource = method.getAnnotation(UseDataSource.class);
        UseDataSource classDataSource = joinPoint.getTarget().getClass().getAnnotation(UseDataSource.class);

        if (methodDataSource == null && classDataSource == null) {
            return joinPoint.proceed();
        }

        if (methodDataSource != null) {
            // Invoking method with method annotation declared dataSource.
            return proceedWithDataSource(joinPoint, methodDataSource.value());
        }

        // Invoking method with class annotation declared dataSource.
        return proceedWithDataSource(joinPoint, classDataSource.value());
    }

    private Object proceedWithDataSource(ProceedingJoinPoint joinPoint, DataSourceEnum dataSource) throws Throwable {
        DataSourceEnum preDataSource = DataSourceContext.getCurrentDataSource();
        DataSourceContext.use(dataSource);

        try {
            return joinPoint.proceed();
        } finally {
            DataSourceContext.use(preDataSource);
        }
    }
}
```

最后进行数据源切面 `Aspect` 的声明，示例如下：

```java
@Component
@Aspect
public class DataSourceRoutingAspect implements Ordered {

    private final DataSourceRoutingAdvice advice = new DataSourceRoutingAdvice();

    @Override
    public int getOrder() {
        return CONSTANTS.DATA_SOURCE_ROUTING_ASPECT_ORDER;
    }

    @Pointcut("@within(UseDataSource)")
    public void typeHasAnnotation() {}

    @Pointcut("@annotation(UseDataSource)")
    public void methodHasAnnotation() {}

    @Pointcut("(execution(* com.sfc.service..*.*(..)))")
    public void atExecution() {}

    @Around("atExecution() && (typeHasAnnotation() || methodHasAnnotation())")
    public Object aroundRoutingDataSource(ProceedingJoinPoint joinPoint) throws Throwable {
        return advice.aroundRoutingDataSource(joinPoint);
    }
}
```

需要注意的是，这里对切面的顺序进行了指定，这是为了保证数据源切换切面在事务切面之前，保证能在事务切面调用 `DataSource` 的 `getConnection` 方法前进行数据源的切换。指定事务切面的顺序如下，需保证 `DATA_SOURCE_ROUTING_ASPECT_ORDER < TRANSACTION_ASPECT_ORDER` 。

```java
@EnableTransactionManagement(order = CONSTANTS.TRANSACTION_ASPECT_ORDER)
```

在正确的配置了 `DataSourceRoutingAspect` 后，开发者就可以进行声明式的数据源切换了。

```java
@Service
@UseDataSource(DataSourceEnum.DB_1)
public class CheckService {

    public void checkMain() {
        // use DB_1 data source
    }

    @UseDataSource(DataSourceEnum.DB_2)
    public void checkMain() {
        // use DB_2 data source
    }
}
```

## 总结

`AbstractRoutingDataSource` 使用代理数据源的方式支持了多数据源的切换，当与 AOP 结合时，可以实现便捷的多数据源切换方案。然而，受限于 Spring AOP 的动态代理实现原理，该方案也有一定的局限性，如无法对私有方法进行增强，也无法对通过 this 引用的内部方法调用进行增强。同时，在事务场景下，一但进入到事务执行内部，就无法进行数据源的切换，除非事务内调用的方法的事务的传播属性是 `REQUIRES_NEW`。实际使用中，还是应当尽量贯彻 SRP，避免在一次调用中进行数据源的切换。

## Reference

* [Dynamic DataSource Routing with Spring @Transactional](http://fedulov.website/2015/10/14/dynamic-datasource-routing-with-spring/)
* [Dynamic DataSource Routing](https://spring.io/blog/2007/01/23/dynamic-datasource-routing/)
* [Spring Framework Reference, transaction-declarative-applying-more-than-just-tx-advice](https://docs.spring.io/spring/docs/4.3.15.RELEASE/spring-framework-reference/html/transaction.html#transaction-declarative-applying-more-than-just-tx-advice)
* [Spring JPA – Multiple Databases](https://www.baeldung.com/spring-data-jpa-multiple-databases)

