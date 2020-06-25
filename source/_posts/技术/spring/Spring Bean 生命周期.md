---
layout: post
title: Spring Bean 生命周期
categories: Spring
tags: [Spring, Spring Bean]
date: 2020-06-20 21:00:00
description: Spring Bean 生命周期
---

# Spring 的启动流程和 Bean 的生命周期

为了探究一下 Spring 的启动流程和 Bean 的生命周期，这里启动了一个 `Spring Boot` 的 Web MVC 服务 ，从代码出发，分析 Spring 的启动流程和 Bean 的生命周期

## Spring 应用的启动

我们从 `SpringApplication#run` 方法开始入手，在这里回去创建一个 ApplicationContext，并且对 ApplicationContext 进行初始化刷新

![SpringApplication#run](/assets/picture/spring_application_run.png "SpringApplication#run")

在 `SpringApplication#run` 方法中创建 ApplicationContext 的时候，会应用的类型去除去创建对应的 ApplicationContext， 而基于 `Servlet API` 的 Web 应用对应的 ApplicationContext 是 `AnnotationConfigServletWebServerApplicationContext`

![创建 ApplicationContext](/assets/picture/spring_application_create_application_context.PNG "创建 ApplicationContext")

## `ApplicationContext` 的初始化刷新 ———— `refresh()` 方法

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 初始化容器的状态及开始时间
			// 初始化占位符属性配置文件资源
			// 校验必填的占位符属性是否被解析处理
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 刷新或者创建一个 DefaultListableBeanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// BeanFactory 的准备工作，进行一些设置
			// 设置 BeanFacotory 的类加载器
			// 设置 SpEL 表达式解析器
			// 设置文件资源及属性配置加载器
			// 设置需要忽略的依赖
			// 设置 BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext 等被解析的依赖
			// 注册 ApplicationListener 探测器到 Context 中
			// 注册 LoadTimeWeaverAware 探测器到 Context 中
			// 注册 LoadTimeWeaver 设置临时类加载器
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 在 Context 中的 BeanFactory 在初始化之后，进行自定义的修改，例如扫描指定包中或者指定注解修饰的 Bean 
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				// 触发 BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor 对BeanDefinition 进行自定义修改
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 向 BeanFactory 中注册 BeanPostProcessor
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 初始化 MessageSource，用于参数替换和国际化
				initMessageSource();

				// Initialize event multicaster for this context.
				// 从 BeanFactory 获取 ApplicationEventMulticaster，注册到 ApplicationContext 中去，如果没有则新建 SimpleApplicationEventMulticaster
				// ApplicationEventMulticaster 管理一组 ApplicationListener， 并且向这些 ApplicationListener 广播事件消息
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
			    // ApplicationContext 的子类初始化特殊的 Bean
				onRefresh();

				// Check for listener beans and register them.
				// 从 BeanFactory 中获取出所有的 ApplicationListener 并向 ApplicationEventMulticaster 注册
			    // 通过 ApplicationEventMulticaster 向 ApplicationListener 广播早期的事件消息
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 实例化 BeanFactory 中的所有非懒加载的单例对象
				// 1. 为 BeanFactory 注册 ConversionService，ConversionService 类型转化的工具类
				// 2. 注册解析内嵌的字符串的 StringValueResolver
				// 3. 初始化 LoadTimeWeaverAware 
				// 4. 将为加载 LoadTimeWeaverAware 的临时的 ClassLoader 设置为 null
				// 5. 将 Configuration 置为冻结，将所有的 Bean 的定义元数据都缓存起来，不接受后续改动
				// 6. 将所有非懒加载的单例 Bean 都实例化并且初始化
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 完成刷新操作
				// 1. 清理资源加载相关的缓存
				// 2. 初始化 ApplicationContext 中的 LifecycleProcessor，用于管理 Bean 的生命周期，如果没有 LifecycleProcessor 则新建一个 DefaultLifecycleProcessor 用于此初始化动作
				// 3. 通知 LifecycleProcessor 刷新事件，通过 Lifecycle 触发Bean 的启动操作，如 Tomcat web 服务器的启动
				// 4. 发布完成刷新事件给 ApplicationListener
				// 5. 向 LiveBeansView 注册 ApplicationContext
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// ApplicationContext 初始化失败之后，将创建好的 Bean 清除掉
				destroyBeans();

				// 重置 ApplicationContext 的状态
				cancelRefresh(ex);

				// 抛出异常
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

在 `ApplicationContext#refresh()` 方法中，对 `ApplicationContext` 进行了初始化的刷新，整体步骤如下

- prepareRefresh：刷新准备工作，设置状态、加载属性配置、校验必填属性配置
- obtainFreshBeanFactory：创建 BeanFactory，默认值是 DefaultListableBeanFactory
- prepareBeanFactory：BeanFactory 的应用前的准备工作，为 BeanFactory 进行一些基础设置
- postProcessBeanFactory：在 ApplicationContext 的子类中对 BeanFactory 进行后后置处理，例如 ApplicationContext 指定了扫描范围的情况下，对扫描范围内的 Bean 进行扫描，注册 BeanDefinition
- invokeBeanFactoryPostProcessors：调用 BeanFactoryPostProcessor，对初始化的 BeanDefinition 进行自定义修改；注意这里也会扫描加载 xml 或者 @Configuration 中的 Bean 的 BeanDefinition
- registerBeanPostProcessors：从 BeanFactory 的 BeanDefinition 缓存中获取 BeanPostProcessor 注册到 ApplicationContext 中去，注意这里将 ApplicationListenerDetector 放到了 BeanPostProcessor 的队尾
- initMessageSource：初始化 MessageSource，用于参数替换和国际化
- initApplicationEventMulticaster：创建事件广播器 ApplicationEventMulticaster，注册到 ApplicationContext 中去，用于向一组 ApplicationListener 广播事件
- onRefresh：调用 ApplicationContext 子类的初始化特定类型的 Bean
- registerListeners：从 BeanFactory 的 BeanDefinition 缓存中获取 ApplicationListener，交由 ApplicationEventMulticaster管理，并注册到 ApplicationContext 中去 
- finishBeanFactoryInitialization：将所有非懒加载的 Bean 进行实例化、初始化；完成后锁定容器上下文，不再允许改动 Bean 实例
- finishRefresh：结束刷新，通过 LifecycleProcessor 启动自动启动的 Bean

上述流程的具体时序图如下：

![Spring Bean 生命周期时序图](/assets/picture/bean.lifecycle.flow.svg "Spring Bean 生命周期时序图")

### ApplicationContext 初始化刷新过程中的重要组件

- BeanFactoryPostProcessor: 在 BeanFactory 初始化之后进行一些自定义设置
    - BeanDefinitionRegistryPostProcessor: 对 BeanFactoryPostProcessor 的扩展，支持修改 BeanFactory 中初始化的 BeanDefinition
        - ConfigurationClassPostProcessor: BeanDefinitionRegistryPostProcessor 的实现之一，有 BeanDefinitionRegistryPostProcessor 功能的基础上，支持扫描 @Configuration 中的 @Bean

## Bean 从创建到销毁

![Spring Bean 从创建到销毁](/assets/picture/spring_bean_instantiation_flow.png "Spring Bean 从创建到销毁")


### Bean 从创建到销毁过程中的重要组件

在 Bean 从创建到销毁的过程中也有很多重要组件的

- InstantiationAwareBeanPostProcessor
    - postProcessBeforeInstantiation: 中断 Bean 的正常初始化流程，创建一个 Bean 的代理实现替代 Bean 的实例
    - postProcessAfterInitialization: Bean 的实例或者代理实现进一步初始化之后，进行进一步自定义处理，如果是创建 Bean 的代理实现，则Bean 的创建流程到此结束，初始化流程被中断
- SmartInstantiationAwareBeanPostProcessor
    - determineCandidateConstructors: 查找确认 Bean 实例化的构造方法，注意：个别子类视线中有其他逻辑
- AutowiredAnnotationBeanPostProcessor
    - determineCandidateConstructors: SmartInstantiationAwareBeanPostProcessor 的实现，除了查找构造方法之后，还会查找 @Lookup 修饰的方法存放到 BeanDefinition 中去
- CglibSubclassingInstantiationStrategy
    - instantiateWithMethodInjection: 通过 CGLIB 创建 Bean 的代理实现，含callback
- BeanNameAware
    - setBeanName: 为 Bean 实例设置名称
- BeanClassLoaderAware
    - setBeanClassLoader​: 为 Bean 实例设置 ClassLoader
- BeanFactoryAware
    - setBeanFactory​: 为 Bean 设置 BeanFactory
- BeanPostProcessor
    - postProcessBeforeInitialization​: 在 Bean 进行初始化之前进行插入处理
    - postProcessAfterInitialization: 在 Bean 进行初始化之后进行插入处理
- InitializingBean
    - afterPropertiesSet: 在 postProcessBeforeInitialization​ 之后，postProcessAfterInitialization 之前进行 Bean 的初始化
- DefaultSingletonBeanRegistry
    - 维护三个层级的缓存，用于处理 singleton Bean 的循环依赖
        - singleFactories: 三级缓存，保存 Bean 名称和其 ObjectFactory 的映射关系
        - earlySingletonObjects: 二级缓存，保存有其他依赖，被实例化没有初始化的 Bean
        - singletonObjects: 一级缓存，保存最终完成实例化、初始化 Bean 的缓存
- DestructionAwareBeanPostProcessor
    - postProcessBeforeDestruction: 进行 Bean 实例销毁前的插入操作
- DisposableBean
    - destroy​: 在 DisposableBean#destroy​ 之后的销毁前插入操作

## Bean 的生命周期流程图

将上述流程总结如下图：

![Spring Bean 生命周期流程图](/assets/picture/spring_bean_lifecycle_flow.png "Spring Bean 生命周期流程图")



