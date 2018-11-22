---
title: Spring 的依赖注入
date: 2018/11/19 19:42:00
category: Spring
tags: 
- Java
- Spring
---

依赖注入（Depedency Injection, DI）作为 Spring 的核心功能，极大的改变了代码的组织形式。本文将从 DI 的原理入手，分析 Spring 的相关源码，
并对 Field Injection, Setter Injection, Constructor Injection 进行对比，给出一定的实践建议。

## 概念

依赖注入是指对象不去直接创建自己所依赖的其他对象，而仅仅去声明需要依赖的其他对象，Spring 容器会去创建
这些依赖的对象，并将创建好的对象直接注入到其依赖的实例域中。这一过程与正常的创建过程相反，因此也称为控制反转（Inversion of Control, IoC）。


## 源码

Spring 容器直接管理的对象称之为 bean，其加载过程分为读取定义、根据定义加载两部分: 

1. 获得 bean 的定义: BeanFactory 使用 BeanDefinitionReader 加载 BeanDefinition 到 BeanDefinitionRegistry 进行注册。

1. 加载 bean: 调用 BeanFacotry 的 getBean 方法时，根据 BeanDefinition 来加载对应的 Bean。

在加载 `bean` 的过程中，`AbastractBeanFactory` 作为 `BeanFactory` 接口的抽象实现，将 `BeanFactory` 
的 `getBean` 操作委托到了内部的 `doGetBean` 方法。`doGetBean` 内部进行各种逻辑判断（是否注册过、
是否已经初始化过、依赖的 bean 是否已经加载等）后，调用 `creatBean` 进行 `bean` 的创建。`creatBean` 
是一个抽象方法，这里来看一下 `AbstractAutowireCapableBeanFactory` 中的具体实现。
`AbstractAutowireCapableBeanFactory` 的 `createBean` 方法在真正创建前，会先调用 `resolveBeforeInstantiation` 
来处理需要 AOP 增强的 `bean`，如果该方法返回了代理后的 `bean`，则直接返回该 `bean`。

```java
// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
if (bean != null) {
	return bean;
}
```

对于不需要代理的 `bean`，会委托到内部方法 `doCreateBean`。该方法内部，首先会进行 `bean` 的创建
（注入构造器依赖），并将其包装在 `BeanWrapper` 中，之后进行后置处理器的调用、创建中单例的缓存
（解决循环依赖）、初始化（注入其他依赖）、注册销毁方法等。

## Field Indjection vs. Setter Injection vs. Contructor Injection

Spring 支持的依赖注入方式有 Field Indjection、Setter Injection、Contructor Injection 三种。

```java
// Field Injection
@Autowired
private DepedencyA depedencyA;

// Setter Injection
@Autowired
public void setDepedencyB(DepedencyB depedencyB) {
    this.depedencyB = depedencyB;
}

// Constructor Indection
@Autowired
public void NeedDepedencyIndection(DepedencyC depedencyC) {
    this.depedencyC = depedencyC;
}
```

直观来看，Field Indejection 十分简洁，十分符合直觉。是的，只要一个 `@Autowired` 注解放在实例域上
依赖注入就完成了，再也不用去写 Setter 或者有参的 Constructor 了。

但是，真的是越简洁越好么？

Field Injection 最大的问题是对 Spring 的强依赖。Spring 宣传的低侵入性在这里被很好的打破了。没有
Setter 或有参构造器意味着通过 Field Injection 注入的实例域无法通过常规的手段进行初始化，必须要依赖
DI 容器。这就意味着这不是一个纯粹的 POJO 了，这个类彻底放弃了对自身依赖的管理。使用者也往往无法清晰
的知道正确使用这个类所需要的依赖，而 Setter 或 Construtor 可清晰的告知使用者这个类的依赖。

Field Injection 也让我们感觉到注入依赖实在是太容易了，声明下变量，一个注解就搞定了，于是经常导致注入
过多的依赖。当一个类依赖了过多其他的类，往往意味着违反了单一职责原则 (Single Responsibility Principle, SRP)。
这个类的很多责任应该划分到其他类中，而不是在 DI 的便利下增加其责任。

那么 Setter 注入和构造器注入哪种方式更好呢？

Spring 官方目前推荐尽量使用构造器注入。因为构造注入可以注入 `final`域，让依赖更加的不可变，并且能避免依赖为
`null`的情况。而且如果注入了过多的依赖，构造器也会显得臃肿不堪，会提示开发者注意 SRP 原则。另外，Spring 4.3
版本之后，构造器注入的情况下可以省去 `@Autowired` 注解，代码可以变得更加纯粹，与框架依赖更少。

当然，Setter 注入方式也有它的优点，在许多情况下还是需要通过 setter 来进行注入的。比如注入一些可选的依赖，或者
需要在运行时动态改变的依赖。与构造器注入方式相比，setter 注入的方式还能实现循环依赖的注入，当然循环依赖一定不是
好事，还是不要循环依赖的好。


## Reference

* [Spring Framework Reference, The IoC Container](https://docs.spring.io/spring/docs/4.2.x/spring-framework-reference/html/beans.html)
* [Field Dependency Injection Considered Harmful](https://www.vojtechruzicka.com/field-dependency-injection-considered-harmful/)

