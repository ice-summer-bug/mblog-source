---
layout: post
title: Spring MVC Http 请求处理流程
categories: Spring
tags: [Spring, Spring MVC]
date: 2020-12-10 21:00:00
description: Spring MVC Http 请求处理流程
---

# Spring MVC Http 请求处理流程

## 前言

在 2020年，Spring MVC 还是目前业界主流的 MVC 框架，通过 Spring MVC 可以轻松的构建一个完整的 Web 应用程序。这里我们将从代码触发，简单分析一下 HTTP 请求的处理流程。

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

遍历 `DispatcherServlet` 中的 HandlerMapping，获取到第一个 `HandlerMapping` 匹配到的处理执行链 `HandlerExecutionChain`

***TODO: 待完善具体匹配过程......***

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

#### 调用处理器之前的前置操作

- 获取操作 `WebDataBinder` 的方法 
    - 获取当前处理器中 @InitBinder 修饰的方法
    - 获取 @ControllerAdvice 修饰的类中的 @InitBinder 修饰的方法
- 集合上述操作 `WebDataBinder` 的方法生成 `WebDataBinderFactory`

- 获取操作 Model 的方法 
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

##### 处理器参数处理

`HttpMethodArgumentResolver`
`HttpMessageConverter`
`RequestBodyAdvice`
`WebDataBinder`

##### 反射调用处理器方法

##### 处理器返回结果处理

`HttpMethodReturnValueHandler`
`HttpMessageConverter`
`ResponseBodyAdvice`
