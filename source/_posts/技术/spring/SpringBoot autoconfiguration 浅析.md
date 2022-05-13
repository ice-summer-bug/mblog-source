---
layout: post
title: Spring Boot auto-configuraion 浅析
categories: Spring
tags: [Spring, Spring Boot]
date: 2020-04-20 21:00:00
description: Spring Boot auto-configuraion 浅析
---

# Sprint Boot Auto-configuration 浅析

`Spring Boot` 的自动配置是结合各种 `starter` 一起使用的


## `Conditional` 注解

### Class 相关的 `Conditional` 注解

| 注解 | 说明 |
|-|-|
| @ConditionalOnClass | 当 `classpath` 中存在指定的 `Class` 的时候命中条件 |
| @ConditionalOnMissingClass | 当 `classpath` 中不存在指定的 `Class` 的时候命中条件 |

#### `@ConditionalOnClass`: `classpath` 中存在指定的 `Class` 的时候命中条件，可以指定多个 `Class`
    - org.springframework.boot.autoconfigure.condition.ConditionalOnClass#value 通过 Class<?> 指定条件相关的一个或多个 Class
    - org.springframework.boot.autoconfigure.condition.ConditionalOnClass#name 通过类的全限定名指定条件相关的一个或多个 Class

##### 示例:
```java
@Configuration
public class TestConfiguration {

    /**
    * 示例：当 classpath 中存在全限定名为 com.test.OneService 的类的时候命中条件，一个名为 testService1 的 Bean 将会注册到 Spring上下文中去
    */
    @ConditionalOnClass(name = "com.test.OneService")
    @Bean
    public TestService1 testService1() {
        return new TestService1();
    }

    /**
    * 示例：当 classpath 中存在类型为 com.test.OneService 的类的时候命中条件，一个名为 testService2 的 Bean 将会注册到 Spring上下文中去
    */
    @ConditionalOnClass(value = com.test.OneService.class)
    @Bean
    public TestService2 testService2() {
        return new TestService2();
    }


}
```

#### `@ConditionalOnMissingClass`: `classpath` 中不存在指定的 `Class` 的时候命中条件，可以指定多个 `Class`，只支持通过类的全限定名指定 `Class`
    - org.springframework.boot.autoconfigure.condition.ConditionalOnMissingClass#value 通过类的全限定名指定条件相关的一个或多个 `Class`

##### 示例:
```java
@Configuration
public class TestConfiguration {

    /**
    * 示例：当 classpath 中不存在全限定名为 com.test.OneService 的类的时候命中条件，一个名为 testService1 的 Bean 将会注册到 Spring上下文中去
    */
    @ConditionalOnMissingClass(value = "com.test.OneService")
    @Bean
    public TestService1 testService1() {
        return new TestService1();
    }

}
```

### Bean 相关的 `Conditional` 注解

| 注解 | 说明 |
|-|-|
| @ConditionalOnBean | 当 Spring 上下文中存在指定的 bean 的时候命中条件 |
| @ConditionalOnMissingBean | 当 Spring 上下文中不存在指定的 bean 的时候命中条件 |

#### `@ConditionalOnBean`: Spring 上下文中存在指定的 Bean 的时候命中条件，可以是当前的 Spring 上下文，也可以是当前 Spring 上下文的祖先，或者两者兼具
    - org.springframework.boot.autoconfigure.condition.ConditionalOnBean#value 指定命中条件的 bean 的 Class
    - org.springframework.boot.autoconfigure.condition.ConditionalOnBean#type 指定命中条件的 bean 的类的全限定名
    - org.springframework.boot.autoconfigure.condition.ConditionalOnBean#annotation 指定修饰命中条件的 bean 的 Class 的注解，只有上下文中存在被注解修饰的 bean 的时候才能命中条件
    - org.springframework.boot.autoconfigure.condition.ConditionalOnBean#name 指定命中条件的 bean 的名称
    - org.springframework.boot.autoconfigure.condition.ConditionalOnBean#search 查询命中条件的 bean 的策略，可以查询当前 Spring 上下文、当前 Spring 上下文的祖先，或者所有 Spring 上下文
    - org.springframework.boot.autoconfigure.condition.ConditionalOnBean#parameterizedContainer 可能在泛型中包含命中条件中的 bean 的类型的其他类

***如果 @ConditionalOnBean 没有指定 value 和 type 属性时，将使用方法的返回值作为 type***

##### 示例:
```java
@Configuration
public class TestConfiguration {

    /**
    * 示例：当所有的 Spring 上下文中存在名为 oneService, 而且类型为 "com.test.OneService" 的 bean 的时候命中条件，一个名为 testService 的 Bean 将会注册到 Spring上下文中去
    */
    @Bean
    @ConditionalOnBean(name = "oneService", value = com.test.OneService.class)
    public TestService testService() {
        return new TestService();
    }

    /**
    * 示例：当前的 Spring 上下文中存在名为 oneService, 而且类的全限定名为 com.test.OneService 的 bean 的时候命中条件，一个名为 testService1 的 Bean 将会注册到 Spring上下文中去
    */
    @ConditionalOnBean(name = "oneService", type = "com.test.OneService", search = SearchStrategy.CURRENT)
    @Bean
    public TestService testService1() {
        return new TestService();
    }

    /**
    * 示例：当前的 Spring 上下文中存在名为 oneService, 类的全限定名为为 com.test.OneService，而且 OneService 被 OneAnnotation 修饰时命中条件，一个名为 testService2 的 Bean 将会注册到 Spring上下文中去
    */
    @ConditionalOnBean(name = "oneService", type = "com.test.OneService", annotation = com.test.OneAnnotation.class, search = SearchStrategy.CURRENT)
    @Bean
    public TestService testService2() {
        return new TestService();
    }

    /**
    * 示例：当所有的 Spring 上下文中存在类型为 com.learning.spring.service.OneServiceContainer 的 bean，且 OneServiceContainer 类的定义的泛型中包含 TestService 时命中条件，一个名为 testService3 的 Bean 将会注册到 Spring上下文中去
    */
    @Bean
    @ConditionalOnBean(type = "com.learning.spring.service.OneServiceContainer", parameterizedContainer = OneServiceContainer.class)
    public TestService testService3() {
        return new TestService();
    }

}
```

#### `@ConditionalOnMissingBean`: Spring 上下文中不存在指定的 Bean 的时候命中条件，可以是当前的 Spring 上下文，也可以是当前 Spring 上下文的祖先，或者两者兼具；和 `@ConditionalOnBean` 不同的是它还能指定被忽略的类
    - value: 指定命中条件的 bean 的一个或多个 Class
    - type: 指定命中条件的 bean 的一个或多个类的全限定名
    - ignored: 指定查询条件相关的 bean 的需要被忽略的 Class
    - ignoredType: 指定查询条件相关的 bean 的需要被忽略的类的全限定名
    - annotation: 指定修饰命中条件的 bean 的 Class 的注解，只有上下文中存在被注解修饰的 bean 的时候才能命中条件
    - name: 指定命中条件的 bean 的名称
    - search: 查询命中条件的 bean 的策略，可以查询当前 Spring 上下文、当前 Spring 上下文的祖先，或者所有 Spring 上下文
    - parameterizedContainer: 可能在泛型中包含命中条件中的 bean 的类型的其他类

```java
 @Configuration
public class TestConfiguration {

    /**
    * 示例：当所有的 Spring 上下文中不存在类型为 TestService 的 bean 的时候，创建类型为 TestService，名称为 testService1 的 Bean 
    */
    @Bean
    @ConditionalOnMissingBean(value = TestService.class)
    public TestService testService1() {
        return new TestService();
    }

    /**
    * 示例：当所有的 Spring 上下文中不存在类型为 TestService 的 bean 的时候，创建类型为 TestService，名称为 testService1 的 Bean 
    */
    @Bean
    @ConditionalOnMissingBean(value = TestService.class)
    public TestService testService1() {
        return new TestService();
    }


}
```


# 从源码学习原理

![Spring Conditional 类图](/assets/picture/spring.condition.class.png  "Spring Conditional 类图")

## Condition 接口

Condition 接口中只定义了一个方法，通过条件匹配上下文和注解的元数据判断是否命中条件

```java
@FunctionalInterface
public interface Condition {

	/**
	 * 判断条件是否匹配 
     *
	 * @param context 条件匹配上下文
	 * @param metadata 注解元数据信息
	 * @return 返回是否命中条件
	 */
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```


