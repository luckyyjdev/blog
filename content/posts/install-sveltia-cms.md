+++
title = "给 Hugo 博客装上 Sveltia CMS：一个免费的开源后台"
date = 2026-06-14
draft = false
tags = [ "Hugo", "Sveltia CMS", "Cloudflare", "CMS" ]
categories = [ "技术" ]
summary = "在 Hugo 博客上免费安装 Sveltia CMS，并通过 Cloudflare Workers 配置 GitHub OAuth 认证的完整记录"
+++

## 前言

搭建完博客后，最头疼的就是每次写文章都要走一遍本地编辑 → `hugo server` 预览 → `git push` 的流程。虽然命令行很酷，但有时候只是想快速改个错别字，或者用手机/平板也能写写东西。这时候就需要一个后台管理系统了。

## 为什么选 Sveltia CMS

市面上有很多 Git-based CMS，Sveltia CMS 是 Netlify CMS（现 Decap CMS）的现代替代品，由一位资深的 UX 工程师基于 Svelte 重写。相比原版有几个明显优势：

- **体积小**：不到 300KB（gzip 后），原版有 1.5MB
- **速度快**：使用 GitHub GraphQL API 批量获取内容，秒开
- **体验好**：沉浸式暗色模式、触摸支持、批量操作
- **兼容 Decap CMS 配置**：配置格式基本通用，迁移成本低
- **完全免费开源**

## 安装

Sveltia CMS 不需要 npm 安装，直接通过 CDN 引入一个 HTML 文件就行。

在 Hugo 的 `static/` 目录下创建 `admin/index.html`：

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="robots" content="noindex" />
    <title>Sveltia CMS</title>
  </head>
  <body>
    <script src="https://unpkg.com/@sveltia/cms/dist/sveltia-cms.js"></script>
  </body>
</html>
```

然后在同目录创建 `admin/config.yml` 配置文件：

```yaml
backend:
  name: github
  repo: luckyyjdev/blog
  branch: master
  base_url: https://sveltia-cms-auth.xxxx.workers.dev

media_folder: /static/images
public_folder: /images

locale: "zh-CN"

collections:
    - name: posts
    label: 文章
    label_singular: 文章
    folder: /content/posts
    create: true
    format: toml-frontmatter
    extension: md
    slug: "{{slug}}"
    fields:
            - { label: 标题, name: title, widget: string }
            - { label: 发布日期, name: date, widget: datetime, type: date }
            - { label: 草稿, name: draft, widget: boolean, default: true }
            - { label: 标签, name: tags, widget: list }
            - { label: 分类, name: categories, widget: list }
            - { label: 摘要, name: summary, widget: text, required: false }
            - { label: 正文, name: body, widget: markdown }
```

这里有几个关键点：

- \*\*`format: toml-frontmatter`\*\*：因为我的 Hugo 文章使用 TOML 格式前置元数据（`+++`），需要显式指定
- \*\*`locale: "zh-CN"`\*\*：让管理界面显示中文
- \*\*`media_folder` 和 `public_folder`\*\*：图片上传到 `static/images/`，通过 `/images/` 路径访问

## 配置 GitHub OAuth 认证

这是最折腾的一步。Sveltia CMS 需要通过 GitHub 认证才能读写仓库内容。官方提供了一键部署的 Cloudflare Worker 认证服务。

### 1. 部署认证 Worker

官方项目 [`sveltia-cms-auth`](https://github.com/sveltia/sveltia-cms-auth) 是一份 Cloudflare Workers 脚本，点击 GitHub README 里的一键部署按钮，授权 Cloudflare 后即可自动部署。

部署完成后会得到一个 Worker URL，例如 `https://sveltia-cms-auth.xxxx.workers.dev`。

### 2. 注册 GitHub OAuth App

打开 [GitHub OAuth Apps 设置页](https://github.com/settings/applications/new)，创建新应用：

- **Application name**：随意填，比如 `Sveltia CMS`
- **Homepage URL**：填你的博客地址
- **Authorization callback URL**：`https://sveltia-cms-auth.xxxx.workers.dev/callback`

创建后拿到 **Client ID** 和 **Client Secret**。

### 3. 配置 Worker 环境变量

在 Cloudflare Dashboard → Workers → `sveltia-cms-auth` → Settings → Variables 中添加：

| 变量 | 值 |
| --- | --- |
| `GITHUB_CLIENT_ID` | 上一步的 Client ID |
| `GITHUB_CLIENT_SECRET` | 上一步的 Client Secret（记得点 Encrypt） |
| `ALLOWED_DOMAINS` | `www.luckyyj.eu.org` |

### 4. 更新 config.yml

将 Worker URL 填入 `config.yml` 的 `backend.base_url`：

```yaml
backend:
  name: github
  repo: luckyyjdev/blog
  branch: master
  base_url: https://sveltia-cms-auth.xxxx.workers.dev
```

### 5. 提交部署

推送到 GitHub 后，Cloudflare Pages 自动重新构建。访问 `https://你的域名/admin/` 即可看到登录界面，点击 GitHub 登录完成认证。

### 页脚加管理入口

为了访问方便，我在页脚的 `Powered by Hugo & PaperMod` 后面加上了 `· 管理后台` 的链接。由于 PaperMod 是 git submodule，不能直接修改主题文件，需要在 `layouts/_partials/footer.html` 里覆盖主题模板。

### 导航栏加首页

默认 Hugo 的 Logo 已经链到首页了，但习惯上还是想在导航栏显式加个「首页」入口。在 `hugo.toml` 加一条 `[[menu.main]]` 配置即可：

```toml
[[menu.main]]
  identifier = "home"
  name = "首页"
  url = "/"
  weight = 5
```

## 最终效果

现在打开 `https://www.luckyyj.eu.org/admin/`，用 GitHub 登录后就能直接在浏览器里：

- 创建/编辑/删除文章
- 上传图片
- 设置标签和分类
- 管理草稿和发布状态

所有修改会自动提交到 GitHub 仓库，Cloudflare Pages 检测到变更后自动构建部署，一整套流程完全免费。

## 总结

Sveltia CMS 的安装配置整体还算简单，最麻烦的部分是 GitHub OAuth 认证，好在官方提供了 Cloudflare Worker 的一键部署方案。对于 Hugo + Cloudflare Pages 的用户来说，这是一套零成本的 CMS 方案，值得一试。
