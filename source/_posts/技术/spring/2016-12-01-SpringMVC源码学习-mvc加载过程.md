---
layout: post
title: SpringMVC源码学习 —— MVC 配置加载过程
categories: Spring
tags: [Spring, Spring MVC]
date: 2016-12-01 21:00:00
description: 《Spring揭秘》学习系列，MVC 加载过程
---

### SpringMVC 配置加载过程

`Spring` 中加载配置文件的工具类是 `xxxNamespaceHandler`， 解析 SpringMVC 的配置文件，加载默认配置的工具类就是 `MvcNameSpaceHandler`。

下面来看看 `MvcNameSpaceHandler` 的源码：

```java
public class MvcNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		// 注册 <mvc:annotation-driven> 配置标签解析类，具体解析内容见 [AnnotationDrivenBeanDefinitionParser]
		registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		registerBeanDefinitionParser("default-servlet-handler", new DefaultServletHandlerBeanDefinitionParser());
		registerBeanDefinitionParser("interceptors", new InterceptorsBeanDefinitionParser());
		registerBeanDefinitionParser("resources", new ResourcesBeanDefinitionParser());
		registerBeanDefinitionParser("view-controller", new ViewControllerBeanDefinitionParser());
		registerBeanDefinitionParser("redirect-view-controller", new ViewControllerBeanDefinitionParser());
		registerBeanDefinitionParser("status-controller", new ViewControllerBeanDefinitionParser());
		registerBeanDefinitionParser("view-resolvers", new ViewResolversBeanDefinitionParser());
		registerBeanDefinitionParser("tiles-configurer", new TilesConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("freemarker-configurer", new FreeMarkerConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("velocity-configurer", new VelocityConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("groovy-configurer", new GroovyMarkupConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("script-template-configurer", new ScriptTemplateConfigurerBeanDefinitionParser());
		registerBeanDefinitionParser("cors", new CorsBeanDefinitionParser());
	}

}
```

下面是 `AnnotationDrivenBeanDefinitionParser` 中的 `parse` 方法：
```java
@Override
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);

		// 注册 `<mvc:annotation-driven>` 的组件信息
		CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
		// 将 `<mvc:annotation-driven>` 的组建信息 push 到解析配置的上下文的 栈 中去
		parserContext.pushContainingComponent(compDefinition);

		// 获取 `ContentNegotiationManager`, 如果 <mvc:annotation-driven> 中配置了 ContentNegotiationManager 的实现类名，则返回该实现的引用；
		// 如果没有配置，就会生成一个默认的 ContentNegotiationManager
		// ContentNegotiationManager: 1. 根据request 解析出 mediaType； 2. 根据 mediaType 解析出文件后缀名
		RuntimeBeanReference contentNegotiationManager = getContentNegotiationManager(element, source, parserContext);

		// 创建 RequestMappingHandlerMapping 的Bean 定义
		RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
		handlerMappingDef.setSource(source);
		handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		// 默认 order 为0，优先使用这个 HandlerMapping
		handlerMappingDef.getPropertyValues().add("order", 0);
		handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
		String methodMappingName = parserContext.getReaderContext().registerWithGeneratedName(handlerMappingDef);

		// 如果 <mvc:annotation-driven> 标签设置了 enable-matrix-variables 属性，在 RequestMappingHandlerMapping 中添加 removeSemicolonContent 属性 如果要使用 matrix-variales 属性， removeSemicolonContent 必须是 false
		if (element.hasAttribute("enable-matrix-variables")) {
			Boolean enableMatrixVariables = Boolean.valueOf(element.getAttribute("enable-matrix-variables"));
			handlerMappingDef.getPropertyValues().add("removeSemicolonContent", !enableMatrixVariables);
		}
		else if (element.hasAttribute("enableMatrixVariables")) {
			Boolean enableMatrixVariables = Boolean.valueOf(element.getAttribute("enableMatrixVariables"));
			handlerMappingDef.getPropertyValues().add("removeSemicolonContent", !enableMatrixVariables);
		}

		// 读取 <mvc:annotation-driven> 中的 path-matching 属性
		// 读取 path-matching 中的 suffix-pattern、trailing-slash、registered-suffixes-only
	  // 设置 RequestMappingHandlerMapping 的 useSuffixPatternMatch、useTrailingSlashMatch、useRegisteredSuffixPatternMatch
		// 读取 path-matching 中的 path-helper （UrlPathHelper 或其自定义子类的全限定名）
		// 设置 RequestMappingHandlerMapping 的 UrlPathHelper，如果 path-helper 不为空则在 ParserContext 中设置当前 UrlPathHelper 的别名为 mvcUrlPathHelper
		// 读取 path-matching 中的 path-helper
		// 设置 RequestMappingHandlerMapping 中的 pathMatcher 默认为 AntPathMatcher(ant 风格的路径匹配器)，如果 path-helper 不为空则在 ParserContext 中设置当前 UrlPathHelper 的别名为 mvcPathMatcher
		configurePathMatchingProperties(handlerMappingDef, element, parserContext);

		// RequestMappingHandlerMapping 设置 CorsConfiguration(跨域请求配置信息) 并且 将 corsConfigurations 注册到 ParserContext 中
		RuntimeBeanReference corsConfigurationsRef = MvcNamespaceUtils.registerCorsConfigurations(null, parserContext, source);
		handlerMappingDef.getPropertyValues().add("corsConfigurations", corsConfigurationsRef);

		// 读取 path-matching 中的 conversion-service 获取 conversionService 并且注册到 ParserContext 中去
		RuntimeBeanReference conversionService = getConversionService(element, source, parserContext);
		RuntimeBeanReference validator = getValidator(element, source, parserContext);
		RuntimeBeanReference messageCodesResolver = getMessageCodesResolver(element);


		RootBeanDefinition bindingDef = new RootBeanDefinition(ConfigurableWebBindingInitializer.class);
		bindingDef.setSource(source);
		bindingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		bindingDef.getPropertyValues().add("conversionService", conversionService);
		bindingDef.getPropertyValues().add("validator", validator);
		bindingDef.getPropertyValues().add("messageCodesResolver", messageCodesResolver);

		// 获取配置信息中的 MessageConverter， 如果没有配置或者 register-defaults 为 true， 其实 register-defaults 默认为true
		// 也就是说不管你有没有配置 message-converters 属性  annotation-driven 都会为你注册这写 HttpMessageConverter:
		// ByteArrayHttpMessageConverter、StringHttpMessageConverter、ResourceHttpMessageConverter、SourceHttpMessageConverter、AllEncompassingFormHttpMessageConverter
		// 还有一些是否根据当前 classpath 中是否有对应的jar包才会添加的对应的 HttpMessageConverter
		ManagedList<?> messageConverters = getMessageConverters(element, source, parserContext);
		// 获取用户自定义的 HandlerMethodArgumentResolver， 解析方法参数
		ManagedList<?> argumentResolvers = getArgumentResolvers(element, parserContext);
		// 获取用户自定义的 HandlerMethodReturnValueHandler， 处理方法的返回值
		ManagedList<?> returnValueHandlers = getReturnValueHandlers(element, parserContext);
		String asyncTimeout = getAsyncTimeout(element);
		RuntimeBeanReference asyncExecutor = getAsyncExecutor(element);
		ManagedList<?> callableInterceptors = getCallableInterceptors(element, source, parserContext);
		ManagedList<?> deferredResultInterceptors = getDeferredResultInterceptors(element, source, parserContext);

		// 创建 RequestMappingHandlerAdapter 的 bean 定义
		RootBeanDefinition handlerAdapterDef = new RootBeanDefinition(RequestMappingHandlerAdapter.class);
		handlerAdapterDef.setSource(source);
		handlerAdapterDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		handlerAdapterDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
		handlerAdapterDef.getPropertyValues().add("webBindingInitializer", bindingDef);
		handlerAdapterDef.getPropertyValues().add("messageConverters", messageConverters);
		addRequestBodyAdvice(handlerAdapterDef);
		addResponseBodyAdvice(handlerAdapterDef);

		if (element.hasAttribute("ignore-default-model-on-redirect")) {
			Boolean ignoreDefaultModel = Boolean.valueOf(element.getAttribute("ignore-default-model-on-redirect"));
			handlerAdapterDef.getPropertyValues().add("ignoreDefaultModelOnRedirect", ignoreDefaultModel);
		}
		else if (element.hasAttribute("ignoreDefaultModelOnRedirect")) {
			// "ignoreDefaultModelOnRedirect" spelling is deprecated
			Boolean ignoreDefaultModel = Boolean.valueOf(element.getAttribute("ignoreDefaultModelOnRedirect"));
			handlerAdapterDef.getPropertyValues().add("ignoreDefaultModelOnRedirect", ignoreDefaultModel);
		}

		if (argumentResolvers != null) {
			handlerAdapterDef.getPropertyValues().add("customArgumentResolvers", argumentResolvers);
		}
		if (returnValueHandlers != null) {
			handlerAdapterDef.getPropertyValues().add("customReturnValueHandlers", returnValueHandlers);
		}
		if (asyncTimeout != null) {
			handlerAdapterDef.getPropertyValues().add("asyncRequestTimeout", asyncTimeout);
		}
		if (asyncExecutor != null) {
			handlerAdapterDef.getPropertyValues().add("taskExecutor", asyncExecutor);
		}

		handlerAdapterDef.getPropertyValues().add("callableInterceptors", callableInterceptors);
		handlerAdapterDef.getPropertyValues().add("deferredResultInterceptors", deferredResultInterceptors);
		String handlerAdapterName = parserContext.getReaderContext().registerWithGeneratedName(handlerAdapterDef);

		String uriCompContribName = MvcUriComponentsBuilder.MVC_URI_COMPONENTS_CONTRIBUTOR_BEAN_NAME;
		RootBeanDefinition uriCompContribDef = new RootBeanDefinition(CompositeUriComponentsContributorFactoryBean.class);
		uriCompContribDef.setSource(source);
		uriCompContribDef.getPropertyValues().addPropertyValue("handlerAdapter", handlerAdapterDef);
		uriCompContribDef.getPropertyValues().addPropertyValue("conversionService", conversionService);
		parserContext.getReaderContext().getRegistry().registerBeanDefinition(uriCompContribName, uriCompContribDef);

		RootBeanDefinition csInterceptorDef = new RootBeanDefinition(ConversionServiceExposingInterceptor.class);
		csInterceptorDef.setSource(source);
		csInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, conversionService);
		RootBeanDefinition mappedCsInterceptorDef = new RootBeanDefinition(MappedInterceptor.class);
		mappedCsInterceptorDef.setSource(source);
		mappedCsInterceptorDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		mappedCsInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(0, (Object) null);
		mappedCsInterceptorDef.getConstructorArgumentValues().addIndexedArgumentValue(1, csInterceptorDef);
		String mappedInterceptorName = parserContext.getReaderContext().registerWithGeneratedName(mappedCsInterceptorDef);

		// 创建 ExceptionHandlerExceptionResolver 的 Bean 定义
		RootBeanDefinition exceptionHandlerExceptionResolver = new RootBeanDefinition(ExceptionHandlerExceptionResolver.class);
		exceptionHandlerExceptionResolver.setSource(source);
		exceptionHandlerExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		exceptionHandlerExceptionResolver.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
		exceptionHandlerExceptionResolver.getPropertyValues().add("messageConverters", messageConverters);
		exceptionHandlerExceptionResolver.getPropertyValues().add("order", 0);
		addResponseBodyAdvice(exceptionHandlerExceptionResolver);

		String methodExceptionResolverName =
				parserContext.getReaderContext().registerWithGeneratedName(exceptionHandlerExceptionResolver);

		RootBeanDefinition responseStatusExceptionResolver = new RootBeanDefinition(ResponseStatusExceptionResolver.class);
		responseStatusExceptionResolver.setSource(source);
		responseStatusExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		responseStatusExceptionResolver.getPropertyValues().add("order", 1);
		String responseStatusExceptionResolverName =
				parserContext.getReaderContext().registerWithGeneratedName(responseStatusExceptionResolver);

		RootBeanDefinition defaultExceptionResolver = new RootBeanDefinition(DefaultHandlerExceptionResolver.class);
		defaultExceptionResolver.setSource(source);
		defaultExceptionResolver.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		defaultExceptionResolver.getPropertyValues().add("order", 2);
		String defaultExceptionResolverName =
				parserContext.getReaderContext().registerWithGeneratedName(defaultExceptionResolver);

		parserContext.registerComponent(new BeanComponentDefinition(handlerMappingDef, methodMappingName));
		parserContext.registerComponent(new BeanComponentDefinition(handlerAdapterDef, handlerAdapterName));
		parserContext.registerComponent(new BeanComponentDefinition(uriCompContribDef, uriCompContribName));
		parserContext.registerComponent(new BeanComponentDefinition(exceptionHandlerExceptionResolver, methodExceptionResolverName));
		parserContext.registerComponent(new BeanComponentDefinition(responseStatusExceptionResolver, responseStatusExceptionResolverName));
		parserContext.registerComponent(new BeanComponentDefinition(defaultExceptionResolver, defaultExceptionResolverName));
		parserContext.registerComponent(new BeanComponentDefinition(mappedCsInterceptorDef, mappedInterceptorName));

		// Ensure BeanNameUrlHandlerMapping (SPR-8289) and default HandlerAdapters are not "turned off"
	  // 创建默认的组件 BeanNameUrlHandlerMapping、HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter
		MvcNamespaceUtils.registerDefaultComponents(parserContext, source);

		parserContext.popAndRegisterContainingComponent();

		return null;
	}
```
