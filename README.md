# hsiang0117.github.io

个人主页，基于 Hugo + PaperMod 主题，托管于 GitHub Pages。

- 🔗 **线上地址**: https://hsiang0117.github.io/
- 📦 **仓库**: https://github.com/hsiang0117/hsiang0117.github.io

---

## 项目框架

```
.
├── hugo.yaml                # ★ 站点主配置（标题、导航、主题参数）
├── .gitignore
├── .github/workflows/
│   └── deploy.yml           # GitHub Actions 自动部署
│
├── content/                 # ★ 所有内容都在这里（Markdown）
│   ├── about.md             #   关于页
│   ├── archives.md          #   归档页
│   ├── search.md            #   搜索页
│   ├── posts/               #   博客文章
│   │   ├── _index.md        #     文章列表页标题
│   │   └── *.md             #     每篇一文件
│   └── projects/            #   项目展示
│       ├── _index.md        #     项目列表页标题
│       └── *.md             #     每个项目一文件
│
├── themes/PaperMod/         # 主题（git submodule，一般不用动）
├── static/                  # 静态资源（图片、favicon 等放这里）
├── archetypes/              # 模板（hugo new 时的默认 frontmatter）
└── layouts/                 # 自定义布局（一般不用动）
```

---

## 日常操作

### 1. 写新文章

```bash
# 创建文章（自动填入 frontmatter 模板）
hugo new content posts/我的新文章.md

# 然后编辑生成的文件 content/posts/我的新文章.md
```

#### Frontmatter 模板

每个文章开头的一段 YAML 配置：

```yaml
---
title: "文章标题"           # 必填
date: 2026-06-04             # 必填，决定排序
tags: ["标签1", "标签2"]     # 可选，影响标签页
categories: ["分类名"]        # 可选
series: ["系列名"]           # 可选，同系列文章会联动
ShowToc: true                # 是否显示目录
TocOpen: false               # 目录默认展开？
summary: "一句话摘要"         # 可选，列表页显示的摘要
---
```

正文用 Markdown 写，支持代码块、表格、LaTeX 公式等。

### 2. 添加新项目

```bash
hugo new content projects/项目名.md
```

Frontmatter 同上，正文写项目描述。

### 3. 修改"关于我"

编辑 `content/about.md`，直接改正文。

### 4. 修改站点配置

编辑 `hugo.yaml`，主要关注这几个部分：

| 配置区域 | 用途 |
|----------|------|
| `title` | 浏览器标签页标题 |
| `menu.main` | 顶部导航栏 |
| `params.homeInfoParams` | 首页欢迎语 |
| `params.socialIcons` | 底部社交图标 |
| `params.defaultTheme` | `auto` / `light` / `dark` |

### 5. 换头像 / 加图片

把图片放到 `static/` 目录下，然后在文章中引用：

```markdown
![描述](/图片名.png)
```

如果要用 PaperMod 的 profileMode（大头像模式），取消 `hugo.yaml` 中 `profileMode` 的注释，把头像图放 `static/avatar.png`。

---

## 本地预览

```bash
hugo server -D
```

浏览器打开 `http://localhost:1313`，修改内容后**自动刷新**。

---

## 发布上线

```bash
# 三步走
git add .
git commit -m "描述你改了什么"
git push
```

推送后 GitHub Actions 自动构建部署，约 30 秒生效。

---

## 进阶定制

| 需求 | 操作 |
|------|------|
| 加评论系统 | 在 `hugo.yaml` 的 `params` 下配置 Giscus / Disqus |
| 自定义 CSS | 在 `assets/css/extended/` 下创建 `.css` 文件 |
| 自定义域名 | GitHub Settings → Pages → Custom domain |
| 访问统计 | 在 `layouts/partials/extend_footer.html` 加 Google Analytics |
| 更新主题 | `git submodule update --remote themes/PaperMod` |

---

## 维护备忘

- **PaperMod 版本**: 跟随 git submodule，更新命令见上方
- **Hugo 版本**: 当前为 extended v0.162，GitHub Actions 中用 `peaceiris/actions-hugo@v3` 保持一致
- **主题文档**: https://github.com/adityatelange/hugo-PaperMod/wiki
