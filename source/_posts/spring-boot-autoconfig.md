---
title: Spring Boot 自动配置原理
date: 2026-07-05
updated: 2026-07-05
tags:
  - Spring Boot
  - 自动配置
categories:
  - Java
  - Spring
---

![@EnableAutoConfiguration](/java/spring/1650e0e47481e59a.png)

Spring Boot 提供了自动配置，`@SpringBootApplication` 注解中的 `@EnableAutoConfiguration` 注解是 Spring Boot 自动配置的关键。该注解通过读取 `META-INF/spring.factories` 配置文件实现自动配置。
