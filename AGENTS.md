# AGENTS.md

## Project

Hugo blog using PaperMod theme. Chinese content (`zh-cn` locale). Deployed to Cloudflare Pages.

## Quick Commands

```bash
hugo server                    # local dev server at http://localhost:1313
hugo                           # build to public/
hugo new content posts/xx.md   # new post (creates as draft)
hugo mod tidy                  # clean module cache
```

## Post Front Matter

Posts use **TOML** front matter (`+++`). The archetype at `archetypes/default.md` sets `draft: true` by default — flip to `false` to publish.

```toml
+++
title = "Title"
date = 2026-06-14
draft = true
tags = ["tag1"]
categories = ["cat1"]
summary = "One-line summary"
+++
```

## Sveltia CMS

- Admin interface at `/admin/` (via `static/admin/index.html` + `static/admin/config.yml`)
- Uses the CDN version of Sveltia CMS (loaded from `unpkg.com`), no npm install needed
- Posts use `format: toml-frontmatter` — front matter is TOML (`+++`)
- **Authentication**: For local dev, use the local workflow (no OAuth). For production, deploy a GitHub OAuth proxy (e.g., Cloudflare Worker).
- Media uploads go to `static/images/`, served at `/images/`.

## Key Conventions

- Content language is Chinese; write post content in `zh-cn`.
- The PaperMod theme is a **git submodule** (`themes/PaperMod`). Do not edit files inside `themes/` directly — override via Hugo's layout/asset overrides in the root `layouts/` or `assets/` directories.
- `public/` contains generated output and is tracked in git (likely for Cloudflare Pages). Regenerate with `hugo` before committing if content changed.
- `baseURL` is set to `"/"` (relative) in `hugo.toml`.
- JSON output is enabled (`home = ["HTML", "RSS", "JSON"]`) for client-side search.

## File Structure

- `content/posts/` — blog posts
- `hugo.toml` — site config
- `archetypes/default.md` — post template (TOML front matter)
- `themes/PaperMod` — git submodule (do not edit)
- `layouts/`, `assets/`, `data/`, `i18n/`, `static/` — empty by default, use for overrides
- `public/` — generated site output (tracked in git)
