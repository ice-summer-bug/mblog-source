---
layout: post
title: SpringMVC 中的 @ControllerAdvice
categories: Spring
tags: [Spring, Spring MVC]
date: 2016-12-10 21:00:00
description: \@ControllerAdvice 注解到底有什么用？
---

### SpringMVC 中的 @ControllerAdvice

`SpringMVC` 中常用的注解网上有很多介绍，`@ControllerAdvice` 这个注解相对来说少见一点

从名称上就能看出， `@ControllerAdvice` 是用来增强 `@Controller` 的

				使用 `@ControllerAdvice` 注解增强控制器

				带有 `@ControllerAdvice` 注解的类，可以包含 @ExceptionHandler、@InitBinder, 和 `@ModelAttribute` 注解的方法，
				并且这些注解的方法会通过控制器层次应用到所有 `@RequestMapping` 方法中，而不用一一在控制器内部声明。

看到上面这段话，大家有没有一种兴奋感？
在没有 `@ControllerAdvice` 之前， 我们是怎么使用 `@ExceptionHandler` 的呢？ 很简单，我们给 `Controller` 定义父类，在父类中又一个方法，被 `@ExceptionHandler` 修饰的方法，
在这里处理， 从上面这句话来说，我们现在并不需要定义什么父类，只要用 `@ControllerAdvice` 修饰一个类，它就能成为全局异常处理器！

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

我们可以看到 `@ControllerAdvice` 是一个 `@Component`, 它当然能被扫描
除了 `@ExceptionHandler` 之外，还有 `@InitBinder`、`@ModelAttribute` 修饰的方法

通过 `@InitBinder` 可以做到绑定一个

这里如果只是 这三个功能， `@ControllerAdvice` 还略显鸡肋

下面来看看 `Spring 4.2` 以来的新特性 `ResponseBodyAdvice` 和 `RequestBodyAdvice`

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

我们可以看到这里， 我们可以对 `@ResponseBody` 修饰的方法的返回值进行处理
比如我们对返回结果进行国际化处理，

来个简单的例子:

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
