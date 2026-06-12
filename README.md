# hsiang0117.github.io

个人主页，基于 Hugo + PaperMod 主题，托管于 GitHub Pages。**双语站点**：英文为默认语言（根路径），中文在 `/zh/` 下，导航栏右上角可切换。

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
│   ├── archives.md          #   归档页（英文）+ archives.zh.md（中文）
│   ├── search.md            #   搜索页（英文）+ search.zh.md（中文）
│   ├── posts/               #   博客文章
│   │   ├── _index.md        #     列表页标题（+ _index.zh.md 中文）
│   │   ├── 文章名.md         #     英文版
│   │   └── 文章名.zh.md      #     中文版
│   └── projects/            #   项目展示
│       ├── _index.md        #     列表页标题（+ _index.zh.md 中文）
│       ├── 项目名.md         #     英文版
│       └── 项目名.zh.md      #     中文版
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
hugo new content posts/my-new-post.md       # 英文版
hugo new content posts/my-new-post.zh.md    # 中文版
```

**双语规则**：同名文件 `xxx.md` 是英文版（默认语言），`xxx.zh.md` 是中文版，Hugo 自动把两者关联为互译页面。只写一种语言也可以——另一种语言的列表里不会出现该文章。

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

### 3. 修改站点配置

编辑 `hugo.yaml`，主要关注这几个部分：

| 配置区域 | 用途 |
|----------|------|
| `title` | 浏览器标签页标题（英文站；中文站标题在 `languages.zh.title`） |
| `languages.<en|zh>.menu.main` | 各语言的顶部导航栏 |
| `languages.<en|zh>.params.homeInfoParams` | 各语言的首页欢迎语 |
| `params.socialIcons` | 底部社交图标 |
| `params.defaultTheme` | `auto` / `light` / `dark` |

### 4. 换头像 / 加图片

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
