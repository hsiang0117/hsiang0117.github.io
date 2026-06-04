---
title: "go-api-starter"
date: 2026-02-01
tags: ["Go", "API", "模板"]
categories: ["项目"]
ShowToc: true
summary: "一个 Go REST API 启动模板，集成常用中间件和最佳实践。"
---

## go-api-starter

[GitHub](https://github.com/hsiang0117/go-api-starter) | [文档](https://github.com/hsiang0117/go-api-starter#readme)

生产级 Go REST API 项目模板，开箱即用。

### 内置功能

- ✅ **路由** — Gin 框架
- ✅ **配置** — Viper，支持环境变量
- ✅ **日志** — Zerolog，结构化日志
- ✅ **数据库** — GORM + PostgreSQL，自动迁移
- ✅ **认证** — JWT 中间件
- ✅ **测试** — testify + httptest
- ✅ **Docker** — 多阶段构建
- ✅ **CI/CD** — GitHub Actions

### 项目结构

```
.
├── cmd/server/main.go
├── internal/
│   ├── config/
│   ├── handler/
│   ├── middleware/
│   ├── model/
│   └── repository/
├── migrations/
├── Dockerfile
└── docker-compose.yml
```

### 使用

```bash
git clone https://github.com/hsiang0117/go-api-starter my-api
cd my-api
make dev
```
