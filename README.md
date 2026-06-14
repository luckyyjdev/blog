# 我的博客

基于 [Hugo](https://gohugo.io/) 和 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题搭建的个人技术博客。
访问地址：https://www.luckyyj.eu.org/

## 功能特点

- 📱 响应式设计，适配各种设备
- 🌙 支持暗色/亮色主题自动切换
- 🔍 内置全文搜索功能
- 📖 阅读时间估算
- 📑 自动生成文章目录
- 💬 代码高亮与一键复制
- 🌐 中文内容支持

## 技术栈

- **Hugo** - 静态网站生成器
- **PaperMod** - 简洁优雅的 Hugo 主题
- **Cloudflare Pages** - 静态网站托管

## 本地开发

### 环境要求

- [Hugo](https://gohugo.io/installation/) (extended edition)

### 快速开始

```bash
# 克隆仓库
git clone --recurse-submodules https://github.com/luckyyjdev/blog.git
cd blog

# 启动本地服务器
hugo server

# 访问 http://localhost:1313 预览网站
```

## 写文章

```bash
# 创建新文章
hugo new content posts/my-new-post.md
```

文章使用 TOML 前置元数据：

```toml
+++
title = "文章标题"
date = 2026-06-14
draft = true  # 改为 false 发布
tags = ["标签1", "标签2"]
categories = ["分类"]
summary = "文章摘要"
+++
```

## 项目结构

```
blog/
├── archetypes/      # 文章模板
├── content/         # 博客内容
│   └── posts/       # 文章目录
├── layouts/         # 布局覆盖
├── static/          # 静态资源
├── themes/PaperMod/ # 主题 (git submodule)
└── hugo.toml        # 站点配置
```

## 部署

本博客使用 [Cloudflare Pages](https://pages.cloudflare.com/) 自动部署。推送到 `main` 分支后会自动构建和发布。

## 许可证

MIT License