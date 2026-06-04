---
title: "json-log-viewer"
date: 2026-03-10
tags: ["Go", "CLI", "开源"]
categories: ["项目"]
ShowToc: true
summary: "一个终端 JSON 日志查看器，支持过滤、高亮和实时追踪。"
---
## json-log-viewer

[GitHub](https://github.com/hsiang0117/json-log-viewer) | [文档](https://github.com/hsiang0117/json-log-viewer#readme)

一个用 Go 编写的终端 JSON 日志查看器，类似 `jq` + `tail -f` 的结合体。

### 功能

- 🔍 **字段过滤** — 只显示你关心的 JSON 字段
- 🎨 **语法高亮** — 按日志级别着色
- 📡 **实时追踪** — 类似 `tail -f`，自动刷新
- 🗂️ **多文件支持** — 同时查看多个日志源

### 快速开始

```bash
go install github.com/hsiang0117/json-log-viewer@latest

# 查看日志文件
jlv app.log

# 实时追踪
jlv -f app.log

# 只显示特定字段
jlv -fields time,level,msg app.log
```

### 技术栈

- Go 1.22+
- [bubbletea](https://github.com/charmbracelet/bubbletea) — TUI 框架
- [lipgloss](https://github.com/charmbracelet/lipgloss) — 终端样式
