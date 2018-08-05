---
layout: post
title: Spring揭秘读书笔记 —— IOC BeanFactory
categories: Spring
tags: [Spring,《Spring揭秘》]
date: 2016-07-15 21:00:00
description: 《Spring揭秘》学习系列，BeanFactory 浅谈
---

### BeanFactory

Spring 的Ioc 容器除了是 Ioc Service Provider 还提供了其他的功能， 这边笔记将介绍 Ioc 容器的 Ioc 相关支持以及衍生的高级特性

Spring 中提供两种IOC 容器， `BeanFactory` 和  `ApplicationContext`

##### `BeanFactory`:

基本类型的ICO 容器， 提供完整的IOC支持。 默认采用延迟初始化策略(lazy-load) 只有客户端对象需要访问容器中某个收管理的对象的时候， 才对该受管理的对象进行初始化以及依赖注入的操作。

##### `ApplicationContext`

留待下章讲解


先看看 `BeanFactory` 的定义

```java
public interface BeanFactory {

	/**
	 * Used to dereference a FactoryBean instance and distinguish it from beans created by the FactoryBean.
   * 这个前缀用于区分FactoryBean， 当想从BeanFactory 中获取一个FactoryBean 对象的时候，会返回这个工厂类
   * For example, if the bean named  myJndiObject is a FactoryBean, getting  myJndiObject will return the factory, not the instance returned by the factory.
	 */
	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	/**
	 * @param name the name of the bean to retrieve
	 * @param args arguments to use when creating a bean instance using explicit arguments
	 * (only applied when creating a new instance as opposed to retrieving an existing one)
	 * @since 2.5
	 */
	Object getBean(String name, Object... args) throws BeansException;

	/**
	 * @param args arguments to use when creating a bean instance using explicit arguments
	 * @since 4.1
	 */
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);

}
```

![](/assets/picture/beanDefinition.png "BeanDefinition 类图")

讲到了 BeanFactory 不得不提 BeanDefinitionRegister

`Spring IOC` 中的 `BeanDefinition` 封装了一个被管理的Bean 的所有信息， 再通过 `BeanDefinitionRegister` 将 Bean 注册到 IOC 容器中去



本章章节比较易懂， 这里主要讲讲没这么提及的FactoryBean。

##### `FactoryBean` :

`FactoryBean` 从命名上看跟  `BeanFactory` 很容易混淆。 `FactoryBean` 是 Spring 提供的一种可以扩展容器对象实例化逻辑的接口， 这个命名主语是Bean，定语是Factory； 也就是说它是 Spring 管理的一个普通的Bean， 只是它相对于生产对象来说，它是一个工厂。

```xml
<bean class="com.liam.learn.ioc.factorybean.DateWithFactoryBean" id="dateWithFactoryBean">
    <property name="dateTimeFormatter">
        <ref bean="dateTimeFormatter" />
    </property>
</bean>

<bean id="dateTimeFormatter" class="org.springframework.format.datetime.standard.DateTimeFormatterFactoryBean"/>
```


获取FactoryBean 的方法

```java
public static void main(String[] args) {
    ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring-config.xml");
    DateTimeFormatterFactoryBean dateTimeFormatter = (DateTimeFormatterFactoryBean) applicationContext.getBean("&dateTimeFormatter");
}
```

`BeanFactory` 中的 `getObjectForBeanInstance` 方法

```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
    // 如果 bean 的名称以'&'开头 并且 bean的类型不属于 FactoryBean 将抛出异常
		if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
    // 如果当前 Bean 的引用类型不是 FactoryBean 或者 bean 的名称以 '&' 开头直接返回这个引用
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {  // 从缓存中获取FactoryBean 的引用
			object = getCachedObjectForFactoryBean(beanName);
		}
    // 如果缓存中没有这个FactoryBean 的引用； 将新建引用，存到缓存中，并返回
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```
