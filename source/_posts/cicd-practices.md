---
title: CI/CD 实践
date: 2026-07-05
updated: 2026-07-05
tags:
  - CI/CD
  - GitHub Actions
  - Jenkins
  - Docker
categories:
  - 工程实践
---

### 1.1 CI/CD 流水线概念

```
代码提交 → [ 构建 → 单测 → 扫描 → 集成测试 → 制品 ] → 部署
           └─────────── CI ──────────────┘         └── CD ──┘
```
CI：代码提交后自动构建/测试，10 分钟反馈；CD（交付）：CI 通过自动打包，手动部署；CD（部署）：自动部署到生产。

### 1.2 GitHub Actions 示例

```yaml
`
