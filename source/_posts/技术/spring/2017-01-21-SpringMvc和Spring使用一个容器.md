---
layout: post
title: SpringMVC 容器 和 Spring 容器
categories: Spring
tags: [Spring, Spring MVC]
date: 2017-01-21 21:00:00
description: Spring 两个容器之间的关系
---

##### 前言

在使用 `Spring MVC` 的过程中，遇到一个问题，在 `Spring 容器` 中注册的属性文件， 在 `SpringMVC 容器` 中无法用 `@Value` 标签引入！
按理说， `SpringMVC 容器` 应该能继承父容器中的所有Bean， 为什么不能使用父容器中引入的配置文件信息呢？


*很遗憾，这个问题暂时没能找到答案，如果哪位大神能解答，请告知*

但是问题还是要解决的， 我们只能换个思路解决。 如果 `SpringMVC 子容器` 和 `Spring 父容器` 如果合一了，不就不存在需要重复引入的问题吗？
那么问题来了：

### SpringMVC 容器和 Spring 容器能合一吗？

先来看看一般我们使用 `SpringMVC` 的 `web.xml` 配置文件，

```xxx
<!DOCTYPE web-app PUBLIC
        "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
        "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
    <display-name>Archetype Created Web Application</display-name>

		<!-- spring 父容器配置文件路径 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/spring-root.xml</param-value>
    </context-param>

    <!-- listener：加载spring 父容器 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
				<!-- spring mvc 子容器配置文件路径，如果不配置默认为 /WEB-INF/*-servlet.xml -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring/spring-mvc.xml</param-value>
        </init-param>
    </servlet>

    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <error-page>
        <error-code>500</error-code>
        <location>/error.jsp</location>
    </error-page>

</web-app>
```

`Tomcat` 在解析 `web.xml` 的时候先后加载 `listener` -> `filter` -> `servlet`
先记载 `ContextLoaderListener`

```java
	/**
	* 覆写 `ServletContextListener` 中的 `contextInitialized` 方法，
	* 实现时，调用父类的 `initWebApplicationContext` 方法
	*/
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
```

```java
/**
	 *
	 * @param servletContext 当前 web 容器上下文
	 * @return spring 容器上下文
	 * @see #ContextLoader(WebApplicationContext)
	 * @see #CONTEXT_CLASS_PARAM
	 * @see #CONFIG_LOCATION_PARAM
	 */
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// 创建一个 WebApplicationContext 实例
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			// 当前创建的spring 父容器注册到 web 容器中， 属性名称是： org.springframework.web.context.WebApplicationContext.ROOT
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```

而 `DispatcherServlet` 继承 `FrameworkServlet`, `FrameworkServlet` 继承 `HttpServletBean`
`HttpServletBean` 继承 `HttpServlet`, 并且提供 `init` 方法

HttpServletBean 中的 `init` 方法调用抽象方法 `initServletBean`

```java
@Override
	public final void init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}

		// ......
		// Let subclasses do whatever initialization they like.
		initServletBean();

		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
```

`FrameworkServlet` 实现 `initServletBean` 方法

```java
@Override
	protected final void initServletBean() throws ServletException {
			// ...... 忽略
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
			// ...... 忽略
	}

	protected WebApplicationContext initWebApplicationContext() {
		// 在 web 容器  ServletContext 中查找 属性为 org.springframework.web.context.WebApplicationContext.ROOT 的 WebApplicationContext Spring 父容器
	WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

	// 如果当前以及创建了  SpringMVC 子容器
	if (this.webApplicationContext != null) {
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			// 如果 springmvc 子容器是 ConfigurableWebApplicationContext, 并且没有激活，设置 spring 父容器和 springmvc 子容器之前的父子容器关系
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				if (cwac.getParent() == null) {
					cwac.setParent(rootContext);
				}
				// 配置刷下spring mvc 子容器
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	// 如果当前没有创建 springmvc 子容器
	if (wac == null) {
		// 在 web 容器  ServletContext 中查找 属性为 org.springframework.web.context.WebApplicationContext.ROOT 的 spring 容器， 将这个父容器作为 spring mvc 子容器， 这种情况下， spring mvc 子容器和 spring 父容器就合一了！
		wac = findWebApplicationContext();
	}
	// 如果上一步没有在 web 容器  ServletContext 中查找到 属性为 org.springframework.web.context.WebApplicationContext.ROOT 的 WebApplicationContext Spring 容器；创建 contextConfigLocation 设置的配置文件注定的 springmvc 子容器
	if (wac == null) {
		// No context instance is defined for this servlet -> create a local one
		wac = createWebApplicationContext(rootContext);
	}

	// ...... 忽略

	return wac;
}
```

在这里，我们找到了 两个容器合一的方法！ 果然是 `Time is cheap, show me the code`, 源码能告诉我们一切！
到这里， 找到了配置方式：

```xml
<!DOCTYPE web-app PUBLIC
        "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
        "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
    <display-name>Archetype Created Web Application</display-name>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/spring-root.xml</param-value>
    </context-param>
    <!-- listener -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- spring mvc -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <!-- 该servlet的spring上下文采用WebApplicationContext，不再重复生成上下文 -->
            <param-name>contextAttribute</param-name>
            <param-value>org.springframework.web.context.WebApplicationContext.ROOT</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

-------------------

下面在回头来看看之前没有解决的问题：

### 为什么 `SpringMVC 子容器` 不能继承 `Spring 父容器` 引入的属性文件

*待解决...*
