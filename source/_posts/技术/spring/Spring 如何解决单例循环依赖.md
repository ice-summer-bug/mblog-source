---
layout: post
title: Spring 如何解决单例的循环依赖
categories: Spring
tags: [Spring, Spring Bean]
date: 2020-06-10 21:00:00
description: Spring 如何解决单例的循环依赖
---

# Spring 如何解决单例的循环依赖

## Spring 循环依赖简单示例

Spring 容器中 Bean 的作用范围，最常见的就是下面两种

- `singleton`: Spring 容器范围内的单例
- `prototype`: 原形，每次新建一个新的 bean

而对于单例类型的 Bean，就存在一种循环依赖的情况，如下图

![Spring 循环依赖](/assets/picture/spring_bean_circular_dependencies.svg "Spring 循环依赖")

这里我们就需要对这个循环依赖的情况进行分解，下面我们从代码出发，看看 Spring 中 Bean 的加载流程

## Spring 的 Bean 加载过程代码分析

加载 Bean 的入口是在 BeanFactory 中，如下

```java 
public interface BeanFactory {

	Object getBean(String name) throws BeansException;
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
	Object getBean(String name, Object... args) throws BeansException;
	<T> T getBean(Class<T> requiredType) throws BeansException;
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    // 省略 ......
}
```

下面是具体的 Bean 加载流程

### 从三级缓存中尝试查找 Bean

![BeanFactory#getBean](/assets/picture/bean_factory_get_bean.jpg "BeanFactory#getBean")

`BeanFactory` 获取 Bean 的具体实现是 `BeanFactory#doGetBean` 方法，在这方法中，首先会调用 `DefaultSingletonBeanRegistry#getSingleton` 方法从缓存中获取目标 Bean 的实例

![DefaultSingletonBeanRegistry#getSingleton(java.lang.String)](/assets/picture/bean_factory_get_singleton_from_cache.png "DefaultSingletonBeanRegistry#getSingleton(java.lang.String)")

下面就是 `DefaultSingletonBeanRegistry#getSingleton` 方法的实现

![DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)](/assets/picture/singleton_bean_registry_get_singleton_from_three_cache.png "DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)")


上述代码具体的查找逻辑流程如下：

![从缓存中查找目标 Bean](/assets/picture/get_bean_from_three_singleton_cache.svg "从缓存中查找目标 Bean")

这里我们讲到了 `三级缓存`，三级缓存是 `DefaultSingletonBeanRegistry` 维护的三个 `ConcurrentHashMap`，用于保存单例对象或者它的工厂类实例

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

    /** 一级缓存，Bean 名称对应 Bean 实例的 Map */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    /** 二级缓存，Bean 名称对应 Bean 实例的 Map，一般保存的是依赖其他单例的 Bean */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    /** 三级缓存，Bean 名称对应创建 Bean 的 ObjectFactory 的 Map */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** 当前正在被创建的单例 Bean 的名称集合 */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
}
```

下面的流程中会不断提到这个 `三级缓存`

### 通过自定义的 ObjectFactory 创建目标 Bean 实例 

从 `三级缓存` 中查找不到已经创建好的目标 Bean 的实例之后，接下来会去主动创建目标 Bean 的实例；这里主要是通过 ObjectFactory 去调用 `BeanFactory#createBean` 方法去创建实例

![通过 ObjectFactory 创建单例](/assets/picture/bean_factory_get_singleton_with_object_factory.png "通过 ObjectFactory 创建单例")

创建 Bean 的外围流程是

1. 从一级缓存中获取 Bean 的实例，获取到了自动返回，否则进入下一步
2. 将 Bean 名称放入正在被创建的单例名称集合中
3. 通过 ObjectFactory 去调用 `BeanFactory#createBean` 方法去创建实例
4. 将 Bean 名称从正在被创建的单例名称集合中移除
5. 将单例对象放入一级缓存中

下面是 `BeanFactory#createBean` 的具体逻辑，主要是调用 `doCreateBean` 方法

![通过 ObjectFactory 创建单例](/assets/picture/do_create_bean_with_object_factory.png "通过 ObjectFactory 创建单例")

```java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
        // 省略 ......
		try {
            // 实际创建 Bean 实例
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

### 调用 BeanFactory 创建 Bean 实例

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}
        // 省略 ......
		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}
		// 省略 ......

		return exposedObject;
	}
```

在上面这个实际创建目标 Bean 实例的逻辑中，主要是三个步骤 

1. createBeanInstance: 通过构造函数，反射创建 Bean 的示例，但是没有填充任何属性
2. populateBean: 对 Bean 中的属性进行填充，处理依赖属性
3. initializeBean: 反射调用 afterPropertiesSet 方法或者自定义的 initMethod 进行初始化

***注意，在对Bean 进行了实例化之后，如果配置允许循环依赖，会将只是实例化，但是没有初始化的 Bean 实例放入三级缓存中***

#### createBeanInstance
 
这个过程就是反射调用构造方法，不做赘述，有兴趣的同学可以去看看

#### populateBean: 对 Bean 进行属性填充

这里主要是通过调用 `InstantiationAwareBeanPostProcessor` 的实现，对 Bean 实例中的依赖属性进行注入处理；同时还在处理之前通过 `InstantiationAwareBeanPostProcessor` 进行实例化后的插入操作

![BeanPostProcess 对Bean 进行属性填充](/assets/picture/process_prooperties_with_bean_post_processor.png "BeanPostProcess 对Bean 进行属性填充")

`InstantiationAwareBeanPostProcessor#postProcessProperties` 方法主要是对 Bean 中的属性进行扫描，然后在容器中找到对应的依赖 Bean，进行依赖注入；
`InstantiationAwareBeanPostProcessor` 的是实现主要是 `CommonAnnotationBeanPostProcessor` 和 `AutowiredAnnotationBeanPostProcessor`

##### `AutowiredAnnotationBeanPostProcessor`

`AutowiredAnnotationBeanPostProcessor` 中维护了一个可处理的注解的集合，并且在构造方法中进行初始化

```java
public class AutowiredAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
		implements MergedBeanDefinitionPostProcessor, PriorityOrdered, BeanFactoryAware {

	private final Set<Class<? extends Annotation>> autowiredAnnotationTypes = new LinkedHashSet<>(4);

	public AutowiredAnnotationBeanPostProcessor() {
		this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
}
```

`AutowiredAnnotationBeanPostProcessor#postProcessProperties` 的主要逻辑如下:
首先，它的主体流程

1. 查找 @Autowired、@Value、@javax.inject.Inject 注解修饰的属性和方法
2. 在 BeanFactory 中查找或者创建依赖的 Bean，通过反射进行这些依赖的注入

![AutowiredAnnotationBeanPostProcessor#postProcessProperties](/assets/picture/autowired_meta_data_inject_flow.png "AutowiredAnnotationBeanPostProcessor#postProcessProperties")

下面是 `步骤1` 的具体逻辑

1.1. 查找 @Autowired、@Value、@javax.inject.Inject 注解修饰的属性
1.2. 查找 @Autowired、@Value、@javax.inject.Inject 注解修饰的方法
1.3. 构架依赖注入的元数据
1.4. 将依赖注入的元数据放入缓存

![AutowiredAnnotationBeanPostProcessor#findAutowiringMetadata](/assets/picture/bean_post_processor_build_autowired_meta_data.png "AutowiredAnnotationBeanPostProcessor#findAutowiringMetadata")
![AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata](/assets/picture/bean_post_processor_do_build_autowired_meta_data.png "AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata")
![AutowiredAnnotationBeanPostProcessor#findAutowiredAnnotation](/assets/picture/bean_post_processor_find_autowired_annotation.png "AutowiredAnnotationBeanPostProcessor#findAutowiredAnnotation")

下面是 `步骤2` 的具体逻辑：
 查找 `AutowiredAnnotationBeanPostProcessor` 支持的注解修饰的属性，通过属性的 Class 名称，在 `BeanFactory` 中找到目标 Bean 的实例，或者创建依赖 Bean 的实例，在通过反射，将依赖的 Bean 的实例赋值给这些属性
对于 `AutowiredAnnotationBeanPostProcessor` 支持的注解修饰的方法也是类似的，找到方法参数，通过参数的 Class 名称，在 `BeanFactory` 中找到目标 Bean 的实例，或者创建依赖 Bean 的实例；再将依赖的 Bean 的实例作为参数，通过反射调用注解修饰的方法

找到这些依赖关系之后，再将这些关系缓存起来，下次可以直接使用

![AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject](/assets/picture/autowired_annotation_field_inject.png "AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject")
![AutowiredAnnotationBeanPostProcessor.AutowiredMethodElement#inject](/assets/picture/autowired_annotation_method_inject.png "AutowiredAnnotationBeanPostProcessor.AutowiredMethodElement#inject")

在从 BeanFactory 中查找依赖的 Bean 的时候，如果存在循环依赖，A 和 B 相互依赖的时候，创建 A 的实例时候，实例化 A 之后，通过 BeanPostProcessor 注入 A 的依赖 B 的时候，A 已经被实例化，并且在存在于三级缓存中，这时候对 B 进行依赖注入的时候，会去 BeanFactory 中查找实例 A，从三级缓存中找到实例 A 的 ObjectFactory 之后，得到了实例 A，并将实例 A 从三级缓存中迁移到二级缓存中

![DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)](/assets/picture/bean_factory_get_instantiated_bean.png "DefaultSingletonBeanRegistry#getSingleton(java.lang.String, boolean)")


##### `CommonAnnotationBeanPostProcessor` 

`CommonAnnotationBeanPostProcessor` 和 `AutowiredAnnotationBeanPostProcessor` 类似，也是可以对一些注解修饰的依赖进行解析注入；
`CommonAnnotationBeanPostProcessor` 支持对 `@Resource`、`@javax.xml.ws.WebServiceRef`、`javax.ejb.EJB` 修饰的属性和方法进行扫描手机，整理成依赖信息之后，从 `BeanFactory` 中查找或者创建依赖，再通过反射进行依赖注入；
具体代码逻辑和 `AutowiredAnnotationBeanPostProcessor` 类似，这里不做赘述。

#### initializeBean ———— Bean 的初始化

`BeanFactory#initializeBean` 方法中主要是调用 Bean 的初始化方法，Bean 的初始化方法分为两种

1. `InitializingBean#afterPropertiesSet` 方法
2. BeanDefinition 中的 设置的 init 方法

首先是调用初始化方法的入口

![BeanFactory#initializeBean](/assets/picture/initialize_bean_invoke_init_method.png "BeanFactory#initializeBean")

对于 `InitilalizingBean` 的实现类，调用 `afterPropertiesSet` 方法

![BeanFactory#invokeInitMethods](/assets/picture/bean_factory_do_invoke_init_method.png "BeanFactory#invokeInitMethods")

对于其他类，调用其 BeanDefinition 中设置的 init 方法

![BeanFactory#invokeCustomInitMethod](/assets/picture/initialize_bean_invoke_custom_init_method.png "BeanFactory#invokeCustomInitMethod")

到这里就完成了 Bean 的实例化、依赖注入以及初始化；可以将 Bean 的实例放入一级缓存中


## 循环依赖的解决流程说明

介绍完了 Bean 的创建流程之后，再来看单例循环依赖的解决方案，就是嵌套多层 Bean 创建流程，在这个嵌套过程中，将实例化好的 Bean 放入三级缓存中，具体说明如下：

![Spring 循环依赖](/assets/picture/spring_bean_circular_dependencies.svg "Spring 循环依赖")

在上面这个循环依赖的例子中，Bean A, B, C 的创建过程简要概括如下 

- Bean A 被实例化
- Bean A 的实例 a 对应的 ObjectFactory 会存入第三级缓存 singletonFactories 中，此时 a.b 为 null
- BeanPostProcessor 查找到 Bean A 的依赖 b 
- 从 BeanFactory 中查找 Bean B 的实例，没有找到
- 对 Bean B 进行实例化
- Bean B 的实例 b 对应的 ObjectFactory 会存入第三级缓存 singletonFactories 中，此时 b.c 为 null
- BeanPostProcessor 查找到 Bean B 的依赖 c
- 从 BeanFactory 中查找 Bean C 的实例，没有找到
- 对 Bean C 进行实例化
- Bean C 的实例 c 对应的 ObjectFactory 会存入第三级缓存 singletonFactories 中，此时 c.a 为 null
- BeanPostProcessor 查找到 Bean C 的依赖 a
- 从 BeanFactory 的三级缓存中找到了实例 a 对应的 ObjectFactory，通过 ObjectFactory 获取到了实例 a
- 返回实例 a 的同时，将实例 a 放入二级缓存 earlySingletonObjects 中，并将实例 a 对应的 ObjectFactory 从三级缓存 singletonFactories 中删除
- 通过反射将实例 a 赋值给 c.a
- 对实例 c 进行初始化，并将实例 c 放入一级缓存 singletonObjects 中
- 返回实例 c，通过反射将实例 c 复制给 b.c
- 对实例 b 进行初始化，并将实例 b 放入一级缓存 singletonObjects 中
- 返回实例 b，通过反射将实例 b 复制给 a.b
- 对实例 a 进行初始化，并将实例 a 放入一级缓存 singletonObjects 中


![Spring 循环依赖处理流程](/assets/picture/spring.circular.reference.solving.png "Spring 循环依赖处理流程")




