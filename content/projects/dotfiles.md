---
title: "dotfiles"
date: 2026-01-15
tags: ["shell", "配置", "开源"]
categories: ["项目"]
ShowToc: true
summary: "我的 dotfiles 配置仓库——终端美化、编辑器设置、脚本工具集。"
---

## dotfiles

[GitHub](https://github.com/hsiang0117/dotfiles) | [文档](https://github.com/hsiang0117/dotfiles#readme)

个人 dotfiles，用 [GNU Stow](https://www.gnu.org/software/stow/) 管理。

### 包含内容

| 配置 | 说明 |
|------|------|
| `bash/` | `.bashrc`, `.bash_aliases` |
| `zsh/` | `.zshrc`, Oh My Zsh 插件 |
| `git/` | `.gitconfig`, `.gitignore_global` |
| `vim/` | `.vimrc`, 插件列表 |
| `vscode/` | `settings.json`, `keybindings.json` |
| `scripts/` | 实用脚本合集 |

### 安装

```bash
git clone https://github.com/hsiang0117/dotfiles ~/.dotfiles
cd ~/.dotfiles
stow bash git vim  # 选择性安装
```
