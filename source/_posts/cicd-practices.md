---
title: CI/CD 实践
date: 2026-07-05 12:00:00
updated: 2026-07-05 12:00:00
tags:
  - CI/CD
  - GitHub Actions
  - Jenkins
  - Docker
categories:
  - 工程实践
---

CI/CD 是现代软件工程的基础设施。持续集成（CI）确保代码合并后立即发现集成问题，持续交付/部署（CD）让经过验证的代码自动到达生产环境。

## 一、流水线概念

```
代码提交 → [ 构建 → 单测 → 静态扫描 → 集成测试 → 制品打包 ] → 部署到环境
            └────────── CI ──────────────────────────┘       └── CD ──┘
```

- **CI**：代码 push 后自动触发构建和测试，10 分钟以内给反馈
- **CD（Continuous Delivery）**：CI 通过后自动打包为可部署制品，部署到生产需人工审批
- **CD（Continuous Deployment）**：全自动，CI 通过后直接部署到生产（如 Netflix、Amazon 的实践）

## 二、GitHub Actions 示例

```yaml
name: Build and Deploy
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test -- --coverage
      - run: npm run lint

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t app:${{ github.sha }} .
      - name: Push to registry
        run: |
          docker tag app:${{ github.sha }} registry.example.com/app:latest
          docker push registry.example.com/app:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to K8s
        run: kubectl set image deployment/app app=registry.example.com/app:${{ github.sha }}
```

## 三、分支策略

| 策略 | 做法 | 适用场景 |
|------|------|----------|
| **GitHub Flow** | 只有一个 main 分支，feature 分支合并后立即部署 | 持续部署、SaaS 产品 |
| **Git Flow** | main + develop + feature + release + hotfix | 有固定发布周期的项目 |
| **Trunk-Based** | 所有人在主分支上开发，用 feature flag 控制未完成功能 | 大型团队、每日多次部署 |

## 四、Docker 多阶段构建

```dockerfile
# 构建阶段
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 运行阶段（仅包含运行时依赖）
FROM node:20-slim
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

多阶段构建将编译工具和最终运行环境分离——最终镜像只包含运行时必需的依赖，体积从 1GB 缩小到 200MB。

## 五、部署策略

| 策略 | 做法 | 风险 | 回滚速度 |
|------|------|------|----------|
| **滚动更新** | 逐个替换旧实例 | 低 | 快（分钟级） |
| **蓝绿部署** | 新旧两套环境，切换流量 | 极低 | 即时（秒级） |
| **金丝雀发布** | 先 5% 流量到新版本，逐步扩大 | 极低 | 即时 |

## 六、小结

CI/CD 的核心价值在于**缩短反馈周期**和**降低发布风险**。GitHub Actions 适合中小团队快速上手，Jenkins 适合需要高度定制化的大型组织。配合 Docker 容器化和 K8s 编排，可以实现从代码提交到生产部署的全自动化——这也是所有高绩效工程团队的标配。
