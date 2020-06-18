---
layout: post
title: SpringMVC 中的 @ControllerAdvice
categories: Spring
tags: [Spring, Spring MVC]
date: 2016-12-10 21:00:00
description: Spring MVC 中的 @ControllerAdvice 注解到底有什么用？
---

# Spring MVC 中的 @ControllerAdvice

## `@ControllerAdvice` 简介

`Spring MVC` 中常用的注解网上有很多介绍，`@ControllerAdvice` 这个注解相对来说少见一点；从名称上就能看出， `@ControllerAdvice` 是用来增强 `@Controller` 的

				使用 `@ControllerAdvice` 注解可以增强控制器 Controller，带有 `@ControllerAdvice` 注解的类，可以包含 @ExceptionHandler、@InitBinder, 和 `@ModelAttribute` 注解的方法，
				并且这些注解的方法会通过控制器层次应用到所有 `@RequestMapping` 方法中，而不用一一在控制器内部声明。

## 使用 `@ControllerAdvice` 进行异常处理

看到上面这段话，大家有没有一种兴奋感？
在没有 `@ControllerAdvice` 之前， 我们是怎么使用 `@ExceptionHandler` 的呢？ 很简单，我们给 `Controller` 定义父类，在父类中创建一个方法，被 `@ExceptionHandler` 修饰的方法，

而有了 @ControllerAdvice 之后，我们并不需要定义什么父类，只要用 `@ControllerAdvice` 修饰一个类，它就能成为全局异常处理器了！

                Talk is cheap, show me the code!

先来一个 Controller:

```java
@Controller
@RequestMapping("/spring/demo")
public class BaseController {

    @RequestMapping("/hello")
    @ResponseBody
    public String hello() {
        throw new RuntimeException("test exceptions");
    }
}
```
再来一个 ControllerAdvice, 直接返回一个 json 格式的异常信息

```java
@ControllerAdvice
public class BaseControllerAdvice {

    @ExceptionHandler
    @ResponseBody
    public Object handlerException(Exception e, HttpServletRequest request) {
        logger.info("exception: {}", e);
        return e;
    }
}
```

再来贴一下 spring 配置信息:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

    <mvc:annotation-driven/>

    <context:component-scan base-package="com.springdemo.liam">
        <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
        <context:include-filter type="annotation" expression="org.springframework.web.bind.annotation.ControllerAdvice"/>
        <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Service"/>
    </context:component-scan>
</beans>
```

看看运行结果，很不错，我们实现了全剧异常处理
![图片](/assets/picture/errHandler.png)


到这里大家会比较疑惑，为什么 ControllerAdvice 可以被自动扫描呢？

先来看看源码：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface ControllerAdvice {

	@AliasFor("basePackages")
	String[] value() default {};

	@AliasFor("value")
	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};

	Class<?>[] assignableTypes() default {};

	Class<? extends Annotation>[] annotations() default {};

}
```

我们可以看到 `@ControllerAdvice` 是一个 `@Component`, 它当然能被扫描注册到上下文中

## `@ControllerAdvice` 异常处理实现原理

Spring MVC 中异常处理器中最常用的就是  `ExceptionHandlerExceptionResovler`，它维护了一个 @ControllerAdvice 注解修饰的类 @ExceptionHandler 修饰的异常处理方法的缓存

```java
public class ExceptionHandlerExceptionResolver extends AbstractHandlerMethodExceptionResolver
		implements ApplicationContextAware, InitializingBean {

    // @ControllerAdvice 对应 ExceptionHandlerMethodResolver 的 Map 缓存
	private final Map<ControllerAdviceBean, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache =
			new LinkedHashMap<>();
    
    // 缓存新建默认 HandlerMethodReturnValueHandler 时用到的 ResponseBodyAdvice 列表
	private final List<Object> responseBodyAdvice = new ArrayList<>();
}
```

`exceptionHandlerAdviceCache` 在 `ExceptionHandlerExceptionResovler` 初始化的时候被赋值

![初始化 ExceptionHandlerExceptionResovler 中的 exceptionHandlerAdviceCache](/assets/picture/exception_handler_init_with_controller_advice.png "初始化 ExceptionHandlerExceptionResovler 中的 exceptionHandlerAdviceCache")

在从处理器 Controller 中获取不到 @ExceptionHandler 修饰的异常处理方法后，会从 `exceptionHandlerAdviceCache` 中获取 `ExceptionHandlerExceptionResovler` 再获取到异常处理方法

![从 @ControllerAdvice 缓存中获取 ExceptionHandler](/assets/picture/find_exception_handler_from_controller_advice_cache.png "从 @ControllerAdvice 缓存中获取 ExceptionHandler")

`ExceptionHandlerExceptionResovler` 还会对调用返回值处理器对返回值进行处理，而初始化时候赋值的 `responseBodyAdvice` 会被用来新建默认的 `HandlerMethodReturnValueHandler`，主要是 `RequestResponseBodyMethodProcessor`

![使用 responseBodyAdviceCache 创建 HandlerMethodReturnValueHandler](/assets/picture/init_request_response_body_method_processo_with_response_body_advice.png "使用 responseBodyAdviceCache 创建 HandlerMethodReturnValueHandler")

![调用 HandlerMethodReturnValueHandler 处理返回值](/assets/picture/invoke_return_value_hanlder_after_invoke_exception_handler_method.png "调用 HandlerMethodReturnValueHandler 处理返回值")

在 `RequestResponseBodyMethodProcessor` 中会调用 `ResponseBodyAdvice#beforeBodyWrite` 对返回结果进行插入处理

![调用 ResponseBodyAdvice](/assets/picture/return_value_handler_invoke_message_converter_with_response_body_advice.png "调用 ResponseBodyAdvice")

`ResponseBodyAdvice` 是 `Spring 4.2` 以来的新特性

`ResponseBodyAdvice` 的源码：

```java
public interface ResponseBodyAdvice<T> {

	/**
	 * Whether this component supports the given controller method return type
	 * and the selected {@code HttpMessageConverter} type.
	 * @param returnType the return type
	 * @param converterType the selected converter type
	 * @return {@code true} if {@link #beforeBodyWrite} should be invoked, {@code false} otherwise
	 */
	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

	/**
	 * Invoked after an {@code HttpMessageConverter} is selected and just before
	 * its write method is invoked.
	 * @param body the body to be written
	 * @param returnType the return type of the controller method
	 * @param selectedContentType the content type selected through content negotiation
	 * @param selectedConverterType the converter type selected to write to the response
	 * @param request the current request
	 * @param response the current response
	 * @return the body that was passed in or a modified, possibly new instance
	 */
	T beforeBodyWrite(T body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType,
			ServerHttpRequest request, ServerHttpResponse response);

}
```

下面是使用 @ControllerAdvice 加 ResponseBodyAdvice 的使用效果 ———— 在 ResponseBodyAdvice 中改写返回结果
实际业务中可以进行统一返回值处理，例如国际化处理等等

```java
@ControllerAdvice(annotations = RestController.class)
public class SimpleResponseBodyAdvice implements ResponseBodyAdvice<Result> {

    /**
     * 校验是否是需要的接入点
     * @param returnType
     * @param converterType
     * @return
     */
    public boolean supports(MethodParameter returnType, Class converterType) {
        return returnType.getMethod().getReturnType().equals(Result.class);
    }

    /**
     * 在往 outputStream 中写入返回结果之前，对返回结果进行处理
     */
    public Result beforeBodyWrite(Result body, MethodParameter returnType, MediaType selectedContentType,
                                  Class<? extends HttpMessageConverter<?>> selectedConverterType,
                                  ServerHttpRequest request, ServerHttpResponse response) {
        return body.withRet(false).withData("被 ResponseBodyAdvice 修改的结果");
    }
}

@RestController
public class SimpleRestController {

    @RequestMapping("/test/rest")
    public Result<String> testRest() {
        return Result.success(true, "hello");
    }
}
```

看看结果：

![图片](/assets/picture/responseBodyDemo.png)


## @ControllerAdvice 和 @InitBinder、@ModelAttribute

在 Spring MVC 的组件中 `HandlerAdapter` 负责为请求匹配处理器、调用参数解析器解析参数、通过反射调用处理器、调用返回值处理器处理返回值；
而 `RequestMappingHandleAdapter` 是默认的第一个被调用的 `HanlderAdapter`, 它维护了 @ControllerAdvice 对应的 @InitBinder 的缓存以及 @ControllerAdvice 对应 @ModelAttribute 的缓存

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
    // @ControllerAdvice 修饰的 Bean 和其中 @InitBinder 修饰的方法的映射缓存
	private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache = new LinkedHashMap<>();
    // @ControllerAdvice 修饰的 Bean 和其中 @ModelAttribute 修饰的方法的映射缓存
	private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache = new LinkedHashMap<>();
}
```

`RequestMappingHandleAdapter` 在初始化的时候会填充上述两个缓存

![初始化 @ControllerAdvice 缓存](/assets/picture/init_controller_advice_cache_for_handler_adapter.png "初始化 @ControllerAdvice 缓存")

- 首先查找到 @ControllerAdvice 修饰的 Bean
- 从 @ControllerAdvice Bean 中查找 @InitBinder 修饰的方法，放入 initBinderAdviceCache 中
- 从 @ControllerAdvice Bean 中查找 @ModelAttribute 修饰的方法，放入 modelAttributeAdviceCache 中

![查找 @InitBinder 和 @ModelAttribute放入缓存](/assets/picture/fill_controller_advice_cache_for_handler_adapter.png "查找 @InitBinder 和 @ModelAttribute放入缓存")

`RequestMappingHandleAdapter` 在解析参数、调动处理器之前会做一些前置操作

1. 查找处理器 Controller 中或者 @ControllerAdvice 中 @InitBinder 修饰的方法，构建 WebDataBinderFactory，用于创建 WebDataBinder，在实例化 WebDataBinder 后调用 @InitBinder 修饰的方法对 WebDataBinder 进行自定义设置，而这个 WebDataBinderFactory 会用于调用 @ModelAttribute 修饰的方法，以及调用处理方法的过程中

![创建 WebDataBinderFactory](/assets/picture/web_data_binder_factory_for_handler_adapter.png "创建 WebDataBinderFactory")

2. 调用 `@ControllerAttribute` 修饰的方法，将返回结果放入 Model 中；

![ModelFactory#initModel](/assets/picture/handler_adapter_init_model.png "ModelFactory#initModel")
![ModelFactory#invokeModelAttributeMethods](/assets/picture/handler_adapter_invoke_model_attribute_method.png "ModelFactory#invokeModelAttributeMethods")
![modelMethod.invokeForRequest](/assets/picture/handler_adapter_do_invoke_method_attribute_method.png "modelMethod.invokeForRequest")


3. 调用 `@ControllerAttribute` 修饰的方法时，在 HandlerMethodArgumentResolver 中调用 WebDataBinderFactory 中 @InitBinder 修饰的方法创建 WebDataBiner，进行参数的校验及校验结果处理；在 @InitBinder 修饰的方法中可以对 WebDataBinder 的 Bean 进行自定义的设置

![HandlerMethodArgumentResolver 使用 WebDataBinder](/assets/picture/web_data_binder_for_resolve_argument.png "HandlerMethodArgumentResolver 使用 WebDataBinder")





