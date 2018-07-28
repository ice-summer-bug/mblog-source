---
layout: post
title: Maven 继承方式： scope import
category: 技术
tags: maven
keywords: maven
description: maven 继承方式
---

### 使用 `scope import` 解决Maven 单继承问题

`Maven` 本身支持继承，很多时候我们会创建多模块项目，而多个模块会引入相同的依赖项，这个时候我们就能使用 `Maven` 的父子工程结构，
创建一个父 pom 文件，其他子项目中的 pom 文件继承父pom

```xml
<parent>
  <groupId>base.parent</groupId>-->
	<artifactId>parent.management</artifactId>
	<version>1.0.0.RELEASE</version>
</parent>
```

这样我们就能是的依赖项的管理更加有条理

但是 `Maven` 父子项目结构和 Java继承一样，都是单继承，一个子项目只能制定一个父 pom

很多时候，我们需要打破这种 `单继承`

例如使用 `spring-boot` 的时候， 官方推荐的方式是继承父pom `spring-boot-starter-parent`

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.1.RELEASE</version>
</parent>
```

但是如果项目中已经有了其他父pom， 又想用 `spring-boot` 怎么办呢？

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
    	<groupId>org.springframework.boot</groupId>
    	<artifactId>spring-boot-dependencies</artifactId>
    	<version>1.5.1.RELEASE</version>
    	<scope>import</scope>
    	<type>pom</type>
  </dependencies>
</dependencyManagement>
```

就是使用 `scope import`， 还需要指定 `type pom`

*注意：`scope import` 只能在 `dependencyManagement` 中的使用*
