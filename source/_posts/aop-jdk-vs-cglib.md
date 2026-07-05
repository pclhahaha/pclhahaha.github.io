---
title: Spring AOP — JDK动态代理 vs CGLIB
date: 2026-07-05
updated: 2026-07-05
tags:
  - Spring
  - AOP
  - JDK动态代理
categories:
  - Java
  - Spring
---

CGLIB 是基于字节码生成的代理方式，允许代理扩展一个实体类，无需实现接口。当目标对象没有实现接口时，Spring 会使用 CGLIB 代理。
