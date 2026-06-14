+++
title = "我的博客部署流程"
date = 2026-06-14
draft = false
tags = ["博客", "Hugo", "Cloudflare Pages", "部署"]
categories = ["技术"]
summary = "从零搭建 Hugo 博客并部署到 Cloudflare Pages 的完整流程记录"
+++

## 前言

搭建个人博客是我很久以前想做的事情。动态博客需要云服务器，价格比较高，每年还得续费，我选择白嫖Cloudflare Pages，经过对比Hexo、Hugo等主流静态博客框架后，我选择了 Hugo —— 以其极快的构建速度和灵活的配置脱颖而出。本文记录了从零开始搭建博客并部署到 Cloudflare Pages 的完整流程。

## 环境准备

### 安装 Hugo

macOS 上使用 Homebrew 安装：

```bash
brew install hugo
```

安装完成后验证：

```bash
hugo version
```

### 创建站点

```bash
hugo new site blog
cd blog
```

### 初始化 Git 并添加主题

```bash
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

使用 git submodule 管理主题的好处是后续更新主题非常方便，直接 `git submodule update` 即可。

## 站点配置

编辑 `hugo.toml`，核心配置如下：

```toml
baseURL = "/"
locale = "zh-cn"
title = "我的博客"
theme = "PaperMod"
```

PaperMod 主题提供了丰富的参数配置，我开启了以下功能：

- 自动主题切换（亮色/暗色）
- 文章目录（TOC）
- 阅读时间估算
- 代码高亮与复制按钮
- 全文搜索（通过 JSON 输出）

搜索功能需要在 `outputs` 中启用 JSON：

```toml
[outputs]
  home = ["HTML", "RSS", "JSON"]
```

## 本地开发

```bash
hugo server
```

访问 `http://localhost:1313` 即可实时预览，修改内容后自动热更新。

## 写文章

新建文章：

```bash
hugo new content posts/my-new-post.md
```

Hugo 使用 TOML 前置元数据，文章模板定义在 `archetypes/default.md` 中：

```toml
+++
title = "文章标题"
date = 2026-06-14
draft = true
tags = ["标签"]
categories = ["分类"]
summary = "摘要"
+++
```

默认 `draft: true`，写完后改为 `false` 才会发布。本地预览时加上 `-D` 参数可以查看草稿：

```bash
hugo server -D
```

## 构建站点

```bash
hugo
```

构建产物输出到 `public/` 目录，这就是需要部署的全部静态文件。

## 部署到 Cloudflare Pages

### 为什么选 Cloudflare Pages

- **免费额度充足**：无限站点、无限请求、无限带宽
- **全球 CDN**：自动就近分发，国内访问速度也不错
- **自动部署**：关联 Git 仓库后，push 即部署
- **自定义域名**：支持绑定自己的域名，免费提供 HTTPS

### 部署步骤

1. 将代码推送到 GitHub 仓库
2. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)，进入 Pages
3. 点击 "Create a project"，选择 "Connect to Git"
4. 选择 GitHub 仓库，配置构建设置：
   - **Build command**: `hugo`
   - **Build output directory**: `public`
   - **Hugo version**: 选择最新版本
5. 点击 "Save and Deploy"

### 自定义域名

在 Cloudflare Pages 项目设置中添加自定义域名，按照提示配置 DNS 记录即可。Cloudflare 会自动处理 HTTPS 证书。

## 完整工作流

日常写文章的流程非常简单：

```bash
# 1. 新建文章
hugo new content posts/article-title.md

# 2. 编辑文章内容

# 3. 本地预览
hugo server -D

# 4. 确认无误后，提交推送
git add .
git commit -m "新文章: article-title"
git push
```

推送后 Cloudflare Pages 自动构建部署，很快就可在线访问。


## 总结

整个搭建过程不到一个小时，Hugo 的开发体验非常流畅，Cloudflare Pages 的部署也足够简单。对于想要拥有个人博客的技术人来说，这是一套低成本、高效率的方案。