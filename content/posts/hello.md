---
title: "开张"
date: 2026-11-04T18:00:00+08:00
tags: ["meta", "hugo"]
author: "xscvvxv"
draft: false
---

站点用 Hugo 生成，托管在 GitHub Pages 上。写文章就是往 `content/posts/` 里加一个 Markdown 文件，提 PR，合并后自动构建部署。

<!--more-->

## 写作约定

- 文章目录：`content/posts/`
- frontmatter 固定四个字段：`title` / `date` / `tags` / `author`
- 摘要分隔符：`<!--more-->`
- 发布：开分支 → 提交 PR → 合并进 `main` → Actions 自动构建并部署

## 本地预览（可选）

```bash
hugo server -D
```

访问 <http://localhost:1313/boke/>。