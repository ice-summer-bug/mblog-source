---
layout: post
title: Spring MVC Http 请求处理流程
categories: Spring
tags: [Spring, Spring MVC]
date: 2020-06-01 21:00:00
description: Spring MVC Http 请求处理流程
---

# Spring MVC Http 请求处理流程

## 前言

Spring MVC 还是目前业界主流的 MVC 框架，通过 Spring MVC 可以轻松的构建一个完整的 Web 应用程序。这里我们将从代码触发，简单分析一下 HTTP 请求的处理流程。

## Spring MVC 重要组件

首先介绍一下 Spring MVC 框架中主要组件

- `DispatcherServlet`: Spring MVC 实现 Servlet API 的关键耦合点，管理Spring MVC 的其他组件，是 HTTP 请求处理的控制中心
- `HandlerMapping`: 请求映射器，映射请求和处理器(Handler)，根据请求获取请求处理调用链(HttpExecutionChain)
- `HandlerExecutionChain`: 请求处理调用链，包含一个处理器(Handler)和链状的拦截器(HandlerInterceptor)
- `HandlerAdapter`: 请求适配器，为请求匹配参数解析工具及返回值处理工具，调用处理器 Handler
- `HandlerInterceptor`: Spring MVC 拦截器，提供处理器被调用前后的插入操作，以及请求处理完成后插入操作
- `ViewResolver`: 视图渲染器
- `LocalResolver`: 本地化解析器，可以用于国际化处理
- `View`: 视图，Spring MVC 支持多种视图，jsp, json, xml 等
- `HandlerExceptionResolver`: Http 请求全局异常处理，支持将异常信息转化为视图
- `MultipartResolver`: 判断请求是否是 multipart 请求，将 multipart 请求转化为 Spring MVC multipart 请求，结束请求时清除文件资源缓存


## Spring MVC Http 请求处理流程概览

![Spring MVC Http 请求处理流程](/assets/picture/spring.mvc.flow.png  "Spring MVC http 请求处理流程图")

- DispatcherServlet.doDispatch 接收到 http 请求
- MultipartResolver 判断标记是否是 multipart 请求
- HandlerMapping 根据请求获取包含处理器(Handler) 中的 HandlerExecutionChain 
- 通过处理器(Handler) 获取处理适配器HandlerAdapter
- 链式调用拦截器 HandlerInterceptor 中的 preHandle
- 通过处理适配器调用处理器获取 ModelAndView；分别进行参数注入、处理方法调用、返回结果处理
- 链式调用拦截器中的 postHandle
- 调用 LocalResolver 处理 i18n 信息
- 调用 ViewResolver 解析视图
- 视图渲染
- 链式调用拦截器 afterCompletion
- MultipartResolver 清理文件上传请求处理中的资源占用

***注意：上述流程中请求 ExceptionHandlerInterceptor 会处理请求处理流程中的异常***

## DispatcherServlet 初始化

![DispatcherServlet 类图](/assets/picture/dispatcher_servlet_class_diagram.png  "DispatcherServlet 类图")

`DispatcherServlet` 是 Servlet API 中 Servlet 的具体实现，是 Http 请求处理的控制入口，也是 Spring MVC 其他组件的管理中心

在 `DispatcherServlet` 初始化的时候，会去初始化其他的 MVC 重要组件


```java
	/**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```

如上述代码所述，DispatcherServlet 在初始化的时候会初始化 MultipartResolver、HandlerMapping、HandlerAdapter、HandlerExceptionResolver、ViewResolver 等组件

***注意：`DispatcherServlet` 在 Spring 容器中没有找到部分组件的 Bean 实例的时候，会从 `DispatcherServlet.properties` 文件中加载默认配置，使用默认配置的组件信息***

```properties
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

## Spring MVC Http 请求处理代码简析

### Servlet API 请求入口

`DispatcherServlet` 继承自 `FrameworkServlet`，`FrameworkServlet` 是 `Servlet` 的子类，针对每种 Http 请求有 `doXxx` 方法处理

![FrameworkServlet#processRequest](/assets/picture/frame_servlet_process_request.png  "FrameworkServlet#processRequest")

org.springframework.web.servlet.FrameworkServlet#processRequest

```java
	protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		// 省略
		try {
			doService(request, response);
		}
		catch (ServletException | IOException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (Throwable ex) {
			failureCause = ex;
			throw new NestedServletException("Request processing failed", ex);
		}

		finally {
			// 省略
		}
	}
```

```java
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);
		// 省略
		try {
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}
```

### Http 请求处理核心流程

- `MultipartResolver` 判断是否是 multipart 请求，对 multipart 请求进行标记
- `HandlerMapping` 根据请求获取处理执行链 `HandlerExecutionChain`
- 根据处理器 `Handler` 获取 `HandlerAdapter`
- 链式调用拦截器 `HandlerInterceptor` 中的 `preHandle` 方法
- 调用处理器 `Handler`
- 链式调用拦截器 `HandlerInterceptor` 中的 `postHandle` 方法
- 调用 `ViewResolver` 将 ModelAndView 渲染成视图，或者通过 `HandlerExceptionResolver` 对异常进行处理，返回异常信息的视图
- 链式调用拦截器 `HandlerInterceptor` 中的 `afterCompletion`
- MultipartResolver 清理文件上传请求处理中的资源占用

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
                // 判断请求是否是 multipart 请求
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
                // 通过请求获取请求处理执行链 HandlerExecutionChain
                // 默认的 RequestMappingHandlerMapping 是通过 URL 匹配请求和处理器 Handler
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
                    // 没有匹配的处理器时，直接返回
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
                // 通过处理器获取处理器适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
                // 链式调用拦截器的 preHandle 方法
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
                // 通过处理适配器调用处理器，包含参数解析和返回值处理
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
                // 链式调用拦截器的 postHandle 方法
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
            // 处理请求返回结果，包含异常处理和正常的视图渲染流程
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
            // 链式调用拦截器的 afterCompletion 方法
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
            // 链式调用拦截器的 afterCompletion 方法
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
                // 清理 multipart 请求的处理过程中的资源占用；链式调用拦截器的 afterCompletion 方法
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```



### HandlerMapping 匹配请求和处理器以及拦截器链

遍历 `DispatcherServlet` 中的 HandlerMapping，获取到第一个 `HandlerMapping` 匹配到的处理执行链 `HandlerExecutionChain`

```java
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

#### 主要的 HandlerMapping

在 `DispatcherServlet.properties` 中配置的默认的 HandlerMapping 有

- BeanNameUrlHandlerMapping
	对于name 以 "/" 开头的 Bean, 注册到映射表中，通过 beanName 进行映射匹配
- RequestMappingHandlerMapping
	对于 @Controller 修饰的类，将 @RequestMapping 修饰的方法注册到映射表中，通过 requestMapping 和 URL 进行映射匹配
- RouterFunctionMapping
	自定义 RouteFunction，自定义匹配逻辑，匹配请求和处理器



### HandlerAdapter 调用处理器的过程

在现有版本的 Spring Mvc 中 `HandlerAdapte` 的实现是 `RequestMappingHandlerAdapter`；
`RequestMappingHandlerAdapter` 调用处理器的过程大概如下：

DispatcherServlet#doDispatch ==> AbstractHandlerMethodAdapter#handle ==> RequestMappingHandlerAdapter#handleInternal ==> RequestMappingHandlerAdapter#invokeHandlerMethod

下面是 org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#invokeHandlerMethod 的具体逻辑

```java
 /**
  * Invoke the {@link RequestMapping} handler method preparing a {@link ModelAndView}
  * if view resolution is required.
  * @since 4.2
  * @see #createInvocableHandlerMethod(HandlerMethod)
  */
 @Nullable
 protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
   HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

  ServletWebRequest webRequest = new ServletWebRequest(request, response);
  try {
    // 1. 在处理器类中获取 @InitBinder 注解修饰的方法
    // 2. 在 @ControllerAdvice 修饰的类中，获取 @InitBinder 注解修饰的方法
    // 3. 结合上面两个步骤中的所有 @InitBinder 修饰的方法生成 WebDataBinderFactory
    // DataBinder: 实现参数注入和参数校验，以及绑定结果分析
   WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
   // 1. 在处理器类中获取 @ModelAttribute 注解修饰的方法，注意在方法上 @ModelAttribute 和 @RequestMapping 不能同时使用
   // 2. 在 @ControllerAdvice 修饰的类中，获取 @ModelAttribute 注解修饰的方法
   // 3. 结合上面两个步骤中的所有 @ModelAttribute 修饰的方法生成 ModelFactory
   ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

   // 封装 ServletInvocableHandlerMethod， 设置参数解析器、返回数据处理器、参数名称发现器
   ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
   if (this.argumentResolvers != null) {
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
   }
   if (this.returnValueHandlers != null) {
    invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
   }
   invocableMethod.setDataBinderFactory(binderFactory);
   invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

   ModelAndViewContainer mavContainer = new ModelAndViewContainer();
   mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
   // 初始化 Model，将 session 中的属性参数提取出来合并放入 Model 中
   // 调用 @ModelAttribute 修饰的方法，提取出这些参数属性，放入 Model 中
   modelFactory.initModel(webRequest, mavContainer, invocableMethod);
   mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

   AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
   asyncWebRequest.setTimeout(this.asyncRequestTimeout);

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
   asyncManager.setTaskExecutor(this.taskExecutor);
   asyncManager.setAsyncWebRequest(asyncWebRequest);
   asyncManager.registerCallableInterceptors(this.callableInterceptors);
   asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

   if (asyncManager.hasConcurrentResult()) {
    Object result = asyncManager.getConcurrentResult();
    mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
    asyncManager.clearConcurrentResult();
    LogFormatUtils.traceDebug(logger, traceOn -> {
     String formatted = LogFormatUtils.formatValue(result, !traceOn);
     return "Resume with async result [" + formatted + "]";
    });
    invocableMethod = invocableMethod.wrapConcurrentResult(result);
   }

   invocableMethod.invokeAndHandle(webRequest, mavContainer);
   if (asyncManager.isConcurrentHandlingStarted()) {
    return null;
   }

   return getModelAndView(mavContainer, modelFactory, webRequest);
  }
  finally {
   webRequest.requestCompleted();
  }
 }
```

#### `RequestMappingHandleAdapter` 初始化

`RequestMappingHandleAdapter` 中管理了其他的 MVC 组件，包括 `HandlerMethodArgumentResolver`、`HandlerMethodReturnValueHandler` 等

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
	
	// @ControllerAdvice 修饰的类中的 @ModelAttribute 修饰的方法的缓存
	private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache = new LinkedHashMap<>();

	// @ControllerAdvice 修饰的类中的 @InitBinder 修饰的方法的缓存
	private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache = new LinkedHashMap<>();

	// RequestBodyAdvice 和 ResponseBodyAdvice 的子类实现的缓存列表
	private List<Object> requestResponseBodyAdvice = new ArrayList<>();

	// 一组 HandlerMethodArgumentResolver 的集合，和 方法参数和 HandlerMethodArgumentResolver 的映射关系缓存
	// 处理请求用的 HandlerMethodArgumentResovler
	@Nullable
	private HandlerMethodArgumentResolverComposite argumentResolvers;

	// 专门处理 @InitBinder 修饰的方法的参数的 HandlerMethodArgumentResolverComposite
	// 包含一组 HandlerMethodArgumentResolver 的集合，和 方法参数和 HandlerMethodArgumentResolver 的映射关系缓存
	// 用于处理WebDataBinderFactory
	@Nullable
	private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;

	// 返回结果处理器
	@Nullable
	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
}
```


`RequestMappingHandleAdapter` 初始化的时候会去初始化上述代码中的各种组件，具体代码如下

```java
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		// 1. 
		initControllerAdviceCache();

		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}


	/**
	 * Return the list of argument resolvers to use including built-in resolvers
	 * and custom resolvers provided via {@link #setCustomArgumentResolvers}.
	 */
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		// 新增注解相关的参数解析器，如 @RequestParam，@RequestBody 等
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		// 新增参数类型匹配的参数解析器
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		// 新增自定义参数解析器
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		// 新增兜底参数解析器
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}


	/**
	 * Return the list of argument resolvers to use for {@code @InitBinder}
	 * methods including built-in and custom resolvers.
	 */
	private List<HandlerMethodArgumentResolver> getDefaultInitBinderArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));

		return resolvers;
	}

		/**
	 * Return the list of return value handlers to use including built-in and
	 * custom handlers provided via {@link #setReturnValueHandlers}.
	 */
	private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>();

		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(),
				this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));
		handlers.add(new HttpHeadersReturnValueHandler());
		handlers.add(new CallableMethodReturnValueHandler());
		handlers.add(new DeferredResultMethodReturnValueHandler());
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

		// Annotation-based return value types
		handlers.add(new ModelAttributeMethodProcessor(false));
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		}
		else {
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}
```

- 查找到所有 @ControllerAdvice 修饰的类
- 在 @ControllerAdvice 修饰的类中找到 @ModelAttribute 修饰的方法，放入 modelAttributeAdviceCache 缓存中
- 在 @ControllerAdvice 修饰的类中找到 @InitBinder 修饰的方法，放入 initBinderAdviceCache 缓存中
- 将 RequestBodyAdvice 和 ResponseBodyAdvice 的子类实现放入 requestResponseBodyAdvice 缓存中
- 将默认的参数解析器放入 argumentResolvers 缓存中
	- 将解析注解修饰的参数的解析器放入 argumentResolvers 缓存中，如 @RequestParam，@RequestBody 等
	- 将解析类型匹配的参数解析器放入 argumentResolvers 缓存中，处理指定类型的参数
	- 将自定义参数解析器放入 argumentResolvers 缓存中
	- 将兜底处理的参数解析器放入 argumentResolvers 缓存中
- 将默认的用于解析 @InitBinder 修饰的方法的参数的参数解析器放入 initBinderArgumentResolvers 缓存中
	- 将解析注解修饰的参数的解析器放入 argumentResolvers 缓存中，如 @RequestParam，@RequestBody 等
	- 将解析类型匹配的参数解析器放入 argumentResolvers 缓存中，处理指定类型的参数
	- 将自定义参数解析器放入 argumentResolvers 缓存中
	- 将兜底处理的参数解析器放入 argumentResolvers 缓存中
- 将默认的用于处理返回结果的 HandlerMethodReturnValueHandler 缓存 returnValueHandlers 中
	- 将处理指定类型的返回结果的处理器加入 returnValueHandlers 缓存中
	- 将处理注解修饰的返回结果的处理器加入 returnValueHandlers 缓存中，例如 @ModelAttribute、@ResponseBody
	- 将处理 void 或者字符串或者 Map 类型的返回结果的处理器加入 returnValueHandlers 缓存中
	- 将自定义的返回结果处理器放入 returnValueHandlers 缓存中
	- 将兜底处理的返回结果处理器放入 returnValueHandlers 缓存中

#### 调用处理器之前的前置操作

在调用处理器方法之前需要进行一些准备工作，具体如下：

- 获取操作 `WebDataBinder` 的方法
	- 获取当前处理器中 @InitBinder 修饰的方法
	- 从 initBinderAdviceCache 缓存中获取 @ControllerAdvice 修饰的类中的 @InitBinder 修饰的方法
- 集合上述操作 `WebDataBinder` 的方法生成 `WebDataBinderFactory`
- 获取操作 `Model` 的方法
	- 获取当前处理器方法参数中 @SessionAttribute 修复的方法
	- 获取当前处理器中 @ModelAttribute 修饰的方法
	- 获取 @ControllerAdvice 修饰的类中的 @ModelAttribute 修饰的方法
- 集合上述操作 Model 的方法生成 `ModelFactory`
- 封装 ServletInvocableHandlerMethod， 设置参数解析器、返回数据处理器、参数名称发现器
- 初始化 Model，在 session 中找到 @SessionAtrribute 修饰的方法参数提取出来合并放入 Model 中
- 调用 ModelFactory 中 @ModelAttribute 修饰的方法，提取出这些参数属性，放入 Model 中
- 最终调用处理器 ServletInvocableHandlerMethod#invokeAndHandle

                在这些前置操作中，出现了 MVC 组件 WebDataBinder
                WebDataBinder 支持如下功能
                - 参数字段黑白名单控制
                - 必填参数字段校验
                - 字段加上特定前缀之后，请求中没有该字段时，实现设置字段默认值
	                - 默认 ! 开头的字段为默认值取值字段，可以自定义；
                    - 例如请求中没有 name 参数但是有 !name 字段时，用 !name 字段取值作为 name 属性的默认值
                - 字段加上特定前缀之后，请求中没有该字段时，实现设置字段值清空
	                - 默认 _ 开头的字段为默认值取值字段，可以自定义
	                - 例如请求中没有 name 参数但是有 _name 字段时，取 name 字段数据类型的默认值，如String 为null 赋值给 name

#### 处理器调用过程

`RequestMappingHandlerAdapter` 的实际处理过程就是

- 调用 `HandlerMethodArgumentResolver` 解析参数
- 反射调用处理器方法
- 调用 `HandlerMethodReturnValueHandler` 处理返回结果

##### 处理器参数解析处理

![HandlerMethodArgumentResolver#resolveArgument](/assets/picture/servlet_invocable_handler_method_argument_resolver.png "HandlerMethodArgumentResolver#resolveArgument")

`RequestMappingHandlerAdapter` 会调用包含一组 `HandlerMethodArgumentResolver` 的 `HandlerMethodArgumentResolverComposite` 进行参数解析，下面下介绍一些常用的 `HandlerMethodArgumentResolver`

###### 常用的 `HandlerMethodArgumentResolver`

| HandlerMethodArgumentResolver 名称 | 如何判断是否支持参数解析 | 如何解析参数 | 对应的注解或者参数类型 |
|-|-|-|-|
| RequestParam<br/>MethodArgumentResolver | 1. 解析 @RequestParam 修饰的参数<br/> 2.解析没有注解修饰的，包括基础数据类型、String、<br/>CharSequence、Number、Date、Enum、Date、URL、<br/>URI、Locale、Class 类型及上述类型的数组 | 1. 调用 javax.servlet.ServletRequest<br/>#getParameterValues 获取参数<br/> 2. 调用 DataBinder 进行参数类型转化 | @RequestParam |
| RequestParamMap<br/>MethodArgumentResolver | 1. 解析 @RequestParam 注解修饰的参数 2. 参数类型是 Map，但是 @RequestParam 没有指定 name属性 | 1. 针对 multipart 请求，提取 multipart 请求中的文件 Map <br/> 2. 针对其他请求，通过 javax.servlet.ServletRequest<br/>#getParameterMap 获取参数 Map | @RequestParam |
| RequestPart<br/>MethodArgumentResolver | 1. @RequestPart 修饰的参数<br/> 2. 没有被 @RequestParam 修饰的 Spring multipart 请求 | 1. 通过 MessageConvert 进行参数转换<br/> 2. 调用 RequestResponseBodyAdvice 对参数进行切入处理<br/> 3. 创建 WebDataBinder，进行参数校验并处理校验结果<br/> 4. 为声明为 Optional 类型的可选参数用 Optional 进行封装 | @RequestPart |
| PathVariable<br/>MethodArgumentResolver | 1. @PathVaribale 注解修饰的非 Map 参数 <br/> 2. @PathVariable 注解修饰的, Optional 包装的 Map 参数，且 @PathVariable 没有 value 属性 | HandlerMapping 将 Url 中的参数提取出来，存放到 Http Request 中去，在这里提取出参数Map， 从Map 中提取具体参数 | @PathVariable |
| PathVariableMap<br/>MethodArgumentResolver | @PathVariable 注解修饰的Map 参数，且 @PathVariable 没有 value 属性 | HandlerMapping 将 Url 中的参数提取出来，存放到 Http Request 中去，在这里提取出参数Map | @PathVariable |
| RequestHeader<br/>MethodArgumentResolver | @RequestHeader 修饰的，类型不是Map 的参数 | 从 Http 请求头中提取参数 | @RequestHeader |
| RequestHeaderMap<br/>MethodArgumentResolver | @RequestHeader 修饰的，类型是Map 的参数 | 从 Http 请求头中提取参数，将返回的数组改装成 Map | @RequestHeader |
| SessionAttribute<br/>MethodArgumentResolver | @SessionAttribute 修饰的参数 | 从 HttpSession 中根据名称提取单个参数 | @SessionAttribute |
| RequestAttribute<br/>MethodArgumentResolver | @RequestAttribute 修饰的参数 | 调用 ServletRequest#getAttribute 方法提取单个属性<br/> 服务端 filter 或者 HandlerInterceptor 设置的属性 | @RequestAttribute |
| ServletCookieValue<br/>MethodArgumentResolver | @CookieValue 修饰的参数 | 从 Cookie 中按照名称获取参数 | @CookieValue |
| RequestResponseBody<br/>MethodProcessor | @RequestBody 修饰的参数 | 1. 调用 MessageConverter 提取参数<br/> 2.创建 WebDataBinder 进行参数校验，对校验结果进行处理 <br/> 为声明为 Optional 类型的可选参数用 Optional 进行封装 | @RequestBody |

##### 反射调用处理器方法

![InvocableHandlerMethod#doInvoke](/assets/picture/servlet_invocable_handler_method_reflect_invoke.png  "InvocableHandlerMethod#doInvoke")

##### 处理器返回结果处理

![HandlerMethodReturnValueHandler#handleReturnValue](/assets/picture/servlet_invocable_handler_method_invoke.png "HandlerMethodReturnValueHandler#handleReturnValue")

```java
	/**
	 * Iterate over registered {@link HandlerMethodReturnValueHandler}s and invoke the one that supports it.
	 * @throws IllegalStateException if no suitable {@link HandlerMethodReturnValueHandler} is found.
	 */
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}

	@Nullable
	private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
```

###### 常用的 `HandlerMethodArgumentResolver` ———— `RequestResponseBodyMethodProcessor`
 
 - 如果判定是否支持 `RequestResponseBodyMethodProcessor` 处理
 判断处理器或者处理器的方法是否被 `@ResponseBody` 修饰

- 具体返回值处理过程

					1. 调用 ResponseBodyAdvice#beforeBodyWrite 进行插入处理
					2. 调用 HttpMessageConverter 处理返回结果


### 异常处理

Http 请求处理流程的中的 `请求映射处理器`、`选取处理适配器`、`链式调用拦截器 HandlerInterceptor 中的 preHandle`、`调用处理器`、`链式调用拦截器中的postHandle` 过程被 `try-catch` 包括在内，处理过程中的异常被 `processDispatchResult` 方法处理

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		// 省略 ......
		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				// 省略 ......
				// 通过请求映射处理执行链
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}
				// 省略 ......
				// 链式调用拦截器 HandlerInterceptor 中的 preHandle
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
				// 省略 ......
				// 调用处理器
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
				// 省略 ......
				// 链式调用拦截器中的 postHandle
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			// 省略
		}
	}

	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {
		boolean errorView = false;
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}
		// 省略 ......
	}

	@Nullable
	protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
			@Nullable Object handler, Exception ex) throws Exception {

		request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
		ModelAndView exMv = null;
		if (this.handlerExceptionResolvers != null) {
			for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
				// 调用 HandlerExceptionResolver 将异常解析成 ModelAndView
				exMv = resolver.resolveException(request, response, handler, ex);
				if (exMv != null) {
					break;
				}
			}
		}
		if (exMv != null) {
			if (exMv.isEmpty()) {
				request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
				return null;
			}
			if (!exMv.hasView()) {
				String defaultViewName = getDefaultViewName(request);
				if (defaultViewName != null) {
					exMv.setViewName(defaultViewName);
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using resolved error view: " + exMv, ex);
			}
			else if (logger.isDebugEnabled()) {
				logger.debug("Using resolved error view: " + exMv);
			}
			WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
			return exMv;
		}
		throw ex;
	}
```

#### `HandlerExceptionResolver`

`DispatcherServlet#processHandlerException` 的具体逻辑就是调用 `HandlerExceptionResolver` 进行异常处理

`DispatcherSevlet.properties` 中配置的默认的 `HandlerExceptionResolver` 是

```properties
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```

##### `ExceptionHandlerExceptionResolver` 的异常处理流程

第一个被应用的 `HandlerExceptionResolver` 就是 `ExceptionHandlerExceptionResolver`, 它的核心逻辑是 `ExceptionHandlerExceptionResolver#doResolveHandlerMethodException`

```java
	protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
			HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {

		// 找到处理器(Controller) 中，或者 @ControllerAdvice 修饰的类中， @ExceptionHandler 修饰的异常处理方法，封装成可触发调用的异常处理方法 
		ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
		if (exceptionHandlerMethod == null) {
			return null;
		}
		// 为异常处理方法设置参数解析器
		if (this.argumentResolvers != null) {
			exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		}
		// 为异常处理方法设置返回结果处理器
		if (this.returnValueHandlers != null) {
			exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		}
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
		try {
			if (logger.isDebugEnabled()) {
				logger.debug("Using @ExceptionHandler " + exceptionHandlerMethod);
			}
			Throwable cause = exception.getCause();
			// 调用异常方法处理器，处理过程和调用处理器中的处理方法的流程是一致的
			// 1. 调用参数解析器解析参数
			// 2. 反射调用异常处理方法
			// 3. 调用返回结果处理器，处理返回结果
			if (cause != null) {
				// Expose cause as provided argument as well
				exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, cause, handlerMethod);
			}
			else {
				// Otherwise, just the given exception as-is
				exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, handlerMethod);
			}
		}
		catch (Throwable invocationEx) {
			// Any other than the original exception (or its cause) is unintended here,
			// probably an accident (e.g. failed assertion or the like).
			if (invocationEx != exception && invocationEx != exception.getCause() && logger.isWarnEnabled()) {
				logger.warn("Failure in @ExceptionHandler " + exceptionHandlerMethod, invocationEx);
			}
			// Continue with default processing of the original exception...
			return null;
		}

		if (mavContainer.isRequestHandled()) {
			return new ModelAndView();
		}
		else {
			ModelMap model = mavContainer.getModel();
			HttpStatus status = mavContainer.getStatus();
			ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
			mav.setViewName(mavContainer.getViewName());
			if (!mavContainer.isViewReference()) {
				mav.setView((View) mavContainer.getView());
			}
			if (model instanceof RedirectAttributes) {
				Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
				RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
			}
			return mav;
		}
	}

	protected ServletInvocableHandlerMethod getExceptionHandlerMethod(
			@Nullable HandlerMethod handlerMethod, Exception exception) {

		Class<?> handlerType = null;

		if (handlerMethod != null) {
			handlerType = handlerMethod.getBeanType();
			ExceptionHandlerMethodResolver resolver = this.exceptionHandlerCache.get(handlerType);
			if (resolver == null) {
				// ExceptionHandlerMethodResolver 是查找 @ExceptionHandler 修饰的方法的一个解析器
				// 从处理器方法所在的类中，查找 @ExceptionHandler 修饰的方法，放入 ExceptionHandlerMethodResolver 的缓存中
				resolver = new ExceptionHandlerMethodResolver(handlerType);
				// 将异常处理方法解析器 ExceptionHandlerMethodResolver 放入缓存中
				this.exceptionHandlerCache.put(handlerType, resolver);
			}
			// 调用 ExceptionHandlerMethodResolver 从它的缓存中获取处理器（@Controller）中的方法，直接返回
			Method method = resolver.resolveMethod(exception);
			if (method != null) {
				return new ServletInvocableHandlerMethod(handlerMethod.getBean(), method);
			}
			// 将代理类转化成它的目标类的示例
			if (Proxy.isProxyClass(handlerType)) {
				handlerType = AopUtils.getTargetClass(handlerMethod.getBean());
			}
		}

		// 从缓存重获取 @ControllerAdvice 修饰的类中 @ExceptionHanlder 修饰的方法，包装成 ServletInvocableHandlerMethod 返回 
		// 这里引入在一个问题，@ControllerAdvice 对应 ExceptionHandlerMethodResolver 的缓存 Map 是什么时候被创建的
		for (Map.Entry<ControllerAdviceBean, ExceptionHandlerMethodResolver> entry : this.exceptionHandlerAdviceCache.entrySet()) {
			ControllerAdviceBean advice = entry.getKey();
			if (advice.isApplicableToBeanType(handlerType)) {
				ExceptionHandlerMethodResolver resolver = entry.getValue();
				Method method = resolver.resolveMethod(exception);
				if (method != null) {
					return new ServletInvocableHandlerMethod(advice.resolveBean(), method);
				}
			}
		}

		return null;
	}
```

所以异常处理核心逻辑的概要流程如下

![ExceptionHandlerExceptionResolver#doResolveHandlerMethodException](/assets/picture/exception_handler_exception_resovler_flow.svg "ExceptionHandlerExceptionResolver#doResolveHandlerMethodException")

##### `ExceptionHandlerExceptionResolver` 的初始化流程

在上面的流程中有几个问题：

1. `exceptionHandlerAdviceCache` 缓存是什么时候加载数据的？
2. 提供给异常处理器的参数解析器 HandlerMethodArgumentResolver 和 HandlerMethodReturnValueHandler 的缓存列表是什么时候初始化的？

下面来看看 `ExceptionHandlerExceptionResolver` 的初始化流程

```java

	public ExceptionHandlerExceptionResolver() {
		this.messageConverters = new ArrayList<>();
		this.messageConverters.add(new ByteArrayHttpMessageConverter());
		this.messageConverters.add(new StringHttpMessageConverter());
		try {
			this.messageConverters.add(new SourceHttpMessageConverter<>());
		}
		catch (Error err) {
			// Ignore when no TransformerFactory implementation is available
		}
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}

	public ExceptionHandlerExceptionResolver() {
		this.messageConverters = new ArrayList<>();
		this.messageConverters.add(new ByteArrayHttpMessageConverter());
		this.messageConverters.add(new StringHttpMessageConverter());
		try {
			this.messageConverters.add(new SourceHttpMessageConverter<>());
		}
		catch (Error err) {
			// Ignore when no TransformerFactory implementation is available
		}
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}

		private void initExceptionHandlerAdviceCache() {
		if (getApplicationContext() == null) {
			return;
		}

		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
		for (ControllerAdviceBean adviceBean : adviceBeans) {
			Class<?> beanType = adviceBean.getBeanType();
			if (beanType == null) {
				throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
			}
			ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(beanType);
			if (resolver.hasExceptionMappings()) {
				this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
			}
			if (ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
				this.responseBodyAdvice.add(adviceBean);
			}
		}

		if (logger.isDebugEnabled()) {
			int handlerSize = this.exceptionHandlerAdviceCache.size();
			int adviceSize = this.responseBodyAdvice.size();
			if (handlerSize == 0 && adviceSize == 0) {
				logger.debug("ControllerAdvice beans: none");
			}
			else {
				logger.debug("ControllerAdvice beans: " +
						handlerSize + " @ExceptionHandler, " + adviceSize + " ResponseBodyAdvice");
			}
		}
	}
```

从代码中可以看到上面问题中提到的缓存数据都是在初始化过程中赋值的，具体流程如下

- 在构造方法初始化提供给 HandlerMethodReturnValueHandler 使用的 HttpMessageConverter
	- 默认的 HttpMessageConverter 有 ByteArrayHttpMessageConverter、StringHttpMessageConverter、SourceHttpMessageConverter、AllEncompassingFormHttpMessageConverter
	- ByteArrayHttpMessageConverter: 解析转化 byte[] 类型参数
	- StringHttpMessageConverter: 解析转化 String 类型参数
	- SourceHttpMessageConverter: 解析转化 SDOMSource、SAXSource、StAXSource、StreamSource、Source 类型参数
- 在初始化方法中加载 @ControllerAdvice 中的 @ExceptionHandler 缓存数据
	- 在容器中查找 @ControllerAdvice 
	- 在 @ControllerAdvice 修饰的类中查找 @ExceptionHandler 修饰的方法
	- 在 @ControllerAdvice 修饰的类中查找 ResponseBodyAdvice 的子类，放入缓存中，供 HandlerMethodReturnValueHandler 使用
- 在初始化方法获取默认的参数解析器和自定义的参数解析器，放入 HandlerMethodArgumentResolverComposite 中
	- 新增基于注解的参数解析器 SessionAttributeMethodArgumentResolver 和 RequestAttributeMethodArgumentResolver
	- 新增解析特定参数类型的参数解析器，ServletRequestMethodArgumentResolver、ServletResponseMethodArgumentResolver、RedirectAttributesMethodArgumentResolver、ModelMethodProcessor
	- 新增自定义的参数解析器
- 在初始化方法获取默认的返回值处理器和自定义的返回值处理器，放入 HandlerMethodReturnValueHandlerComposite 中
	- 新增单一用途的处理指定返回值类型的处理器
		- ModelAndViewMethodReturnValueHandler: 处理返回 ModelAndView 的返回值
		- ModelMethodProcessor: 处理返回 Model 的返回值
		- ViewMethodReturnValueHandler: 处理返回 View 的返回值
		- HttpEntityMethodProcessor: 处理返回 ModelAndView 的返回值
		- ModelAndViewMethodReturnValueHandler: 处理返回 HttpEntity 类型，但是不是 RequestEntity 类型的返回值
	- 新增注解修饰的返回值处理器
		- ModelAttributeMethodProcessor: 处理 @ModelAttribute 修饰的方法的返回值
		- RequestResponseBodyMethodProcessor: 处理 @ResponseBody 修饰的方法的返回值，并且触发 ResponseBodyAdvice 切入处理
	- 新增多用途的返回值处理器
		- ViewNameMethodReturnValueHandler: 处理返回值为void 或者 字符串的方法的返回值
		- MapMethodProcessor: 处理返回 Map 类型方法的返回值
	- 新增自定义的返回值处理器
	- 新增不需要 @ModelAttribute 注解修饰的，但是不是基础类型的返回值的处理器，ModelAttributeMethodProcessor(true)


这里我们能看出来，异常处理的过程中，@ExceptionHandler 修饰的方法的参数和返回值类型都有一定的限制，并且会触发 ResponseBodyAdvice 的插入操作

				@ExceptionHandler 修饰的方法的参数限制
				1. 参数类型支持以下情况
					1.1 @SessionAttribute 修饰的参数
					1.2 @ModelAttribute 修饰的参数
					1.3 WebRequest、ServletRequest、MultipartRequest、HttpSession、PushBuilder、Principal、InputStream、Reader、HttpMethod、TimeZone、Locale、ZoneId 类型的参数
					1.4 ServletResponse、OutputStream、Writer 类型的参数
					1.5 RedirectAttributes 类型的参数
					1.6 Model 类型的参数
					1.7 自定义参数解析器可解析的参数
				2. 返回值类型支持以下情况
					2.1 ModelAndView 类型
					2.2 Model 类型
					2.3 View 类型
					2.4 HttpEntity 类型，但是不是它的子类 RequestEntity
					2.5 @ModelAttribute 修饰的方法的返回值
					2.6 @ResponseBody 修饰的方法的返回值
					2.7 void 或者 CharSequence 类型的返回值
					2.8 Map 类型的返回值
					2.9 自定义的返回值处理器支持的类型
					2.10 没有 @ModelAttribute 修饰的，非基础类型的返回值


## 总结

上述就是 Spring MVC 处理 http 请求的流程概述，从代码简单分析了常用的 MVC 注解的工作原理，也留了一下待完善的部分，以上内容均属个人浅见，如果错漏，欢迎指正

待补充部分： HandlerMapping 如何匹配请求和处理器？