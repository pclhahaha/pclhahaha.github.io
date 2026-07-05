---
title: Spring Bean 完整生命周期
date: 2026-07-05
updated: 2026-07-05
tags:
  - Spring
  - Bean生命周期
categories:
  - Java
  - Spring
---

这是 Spring 容器中两个非常重要的扩展点：

- **BeanFactoryPostProcessor**：在 BeanDefinition 加载完成后、bean 实例化之前，对 BeanFactory 进行后置处理。例如 `ConfigurationClassPostProcessor` 会扫描 `@Configuration` 注解标注的类并完成相应的 bean definition 注册。
- **BeanPostProcessor**：在 bean 实例化之后、初始化前后进行拦截处理，允许对 bean 实例进行修改或包装。
