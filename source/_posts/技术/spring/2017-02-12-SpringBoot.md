---
layout: post
title: Spring Boot 初涉
category: 技术
tags: Spring-Boot
keywords: SpringBoot
description:
---

#### 使用 `Maven` 构建 Spring-Boot 项目

1. 继承 父pmo 构建 Spring-Boot 项目

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

  <!-- ...... the artifactId and groupId of the application -->


  <!-- 继承 spring-boot-starter-parent 使用 spring-boot的最优方法； 也可以不继承这个父pom，使用其他方法 -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.7</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
        <!-- 设置插件，打包jar包 -->
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

  <!-- ...... the artifactId and groupId of the application -->


  <dependencyManagement>
      <dependencies>
          <!-- Override Spring Data release train provided by Spring Boot -->
          <dependency>
              <groupId>org.springframework.data</groupId>
              <artifactId>spring-data-releasetrain</artifactId>
              <version>Fowler-SR2</version>
              <scope>import</scope>
              <type>pom</type>
          </dependency>
          <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-dependencies</artifactId>
              <version>1.5.1.RELEASE</version>
              <type>pom</type>
              <!--  -->
              <scope>import</scope>
          </dependency>
          <dependency>
        			<groupId>org.springframework.boot</groupId>
        			<artifactId>spring-boot-starter-web</artifactId>
              <version></version>
      		</dependency>
      </dependencies>
  </dependencyManagement>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.7</java.version>
	</properties>

	<build>
		<plugins>
			<plugin>
        <!-- 设置插件，打包jar包 -->
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

#### dev-tools

```xml
<dependencies>
	<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <!-- 设置插件，打包jar包 -->
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
			<!-- 使用 dev-tools 需要禁用默认的 excludeDevtools -->
      <configuration>
				<excludeDevtools>false</excludeDevtools>
			</configuration>
    </plugin>
  </plugins>
</build>
```

DevToolsPropertyDefaultsPostProcessor
