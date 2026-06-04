---
title: "使用 Hugo + PaperMod 搭建个人主页"
date: 2026-06-04
tags: ["Hugo", "GitHub Pages", "教程"]
categories: ["技术"]
series: ["网站搭建"]
ShowToc: true
TocOpen: true
summary: "从零开始，用 Hugo 和 PaperMod 主题搭建一个托管在 GitHub Pages 上的免费个人主页。"
---

## 为什么选 Hugo？

[Hugo](https://gohugo.io/) 是用 Go 编写的静态站点生成器，以**构建速度极快**著称。配合 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题，可以获得：

- ⚡ 毫秒级构建
- 🌓 自动明暗主题切换
- 🔍 内置全文搜索
- 📱 完全响应式
- 📝 Markdown 编写内容
- 🆓 GitHub Pages 免费托管

## 安装 Hugo

### Windows

```bash
winget install Hugo.Hugo.Extended
```

### macOS

```bash
brew install hugo
```

### Linux

```bash
sudo apt install hugo  # Debian/Ubuntu
```

## 创建站点

```bash
hugo new site my-site
cd my-site
git init
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

## 配置

编辑 `hugo.yaml`，设置主题和基础参数：

```yaml
baseURL: https://<你的用户名>.github.io/
theme: PaperMod
title: "My Site"
```

## 部署

创建 `.github/workflows/deploy.yml`，用 GitHub Actions 自动构建部署。

每次 `git push`，站点自动更新 🎉

---

## 下一步

1. 自定义配色和字体
2. 添加评论系统（如 Giscus）
3. 配置自定义域名
4. 写更多内容！
