+++
title = "Sveltia CMS后台管理页面无法打开排查记录"
date = 2026-06-24
draft = false
tags = [ "技术", "排故", "博客" ]
categories = [ "技术" ]
summary = "Sveltia CMS后台管理页面无法打开排查记录"
+++

## 问题描述

访问 `https://www.luckyyj.eu.org/admin/` 时，Sveltia CMS 后台白屏无法正常加载。

## 观察到的错误

浏览器控制台报错：

```
admin/:10 GET https://static.cloudflareinsights.com/beacon.min.js/... net::ERR_BLOCKED_BY_CLIENT
```

## 排查过程

1. **查看 `static/admin/index.html`** — 只有 Sveltia CMS 的 CDN 引用，代码本身没有问题。

2. **直接请求线上页面查看源码** — 发现 Cloudflare Pages 自动注入了两个脚本：
   - `/cdn-cgi/scripts/.../rocket-loader.min.js`（Cloudflare Rocket Loader™）
   - `static.cloudflareinsights.com/beacon.min.js`（Cloudflare Insights 统计）

3. **分析 Rocket Loader 影响**：
   - Rocket Loader 会延迟所有 JavaScript 的加载和执行
   - 它会修改 script 标签的 `type` 属性（如 `type="xxxx-text/javascript"`）
   - Sveltia CMS 是单页应用（SPA），依赖正常的脚本加载顺序
   - Rocket Loader 的干预会导致 SPA 无法正常初始化

4. **分析 Insights Beacon 影响**：
   - `ERR_BLOCKED_BY_CLIENT` 是客户端广告拦截器（如 uBlock Origin）拦截了统计脚本
   - 这个脚本仅用于站点统计，不影响 Sveltia CMS 的功能

## 根因

**Cloudflare Rocket Loader™** 延迟并修改了 Sveltia CMS 脚本的加载方式，导致 SPA 初始化失败。

## 解决方案

登录 Cloudflare Dashboard → 域名 `luckyyj.eu.org` → **Speed** → **Optimization** → **Rocket Loader™** → 设为 **Off**。

*注：Cloudflare Insights 的 `ERR_BLOCKED_BY_CLIENT` 是广告拦截器导致的正常现象，不影响功能，无需处理。*
