# boke

Hugo 静态博客，GitHub Pages 托管。

## 写作

新文章放 `content/posts/`，frontmatter 四个字段：

```yaml
---
title: "标题"
date: 2026-11-04T18:00:00+08:00
tags: ["tag1", "tag2"]
author: "xscvvxv"
---
```

## 发布流程

1. 从 `main` 开分支，加/改文章
2. 提交 PR
3. 合并到 `main` → `Deploy Hugo site to Pages` 自动构建并部署

站点地址：<https://xscvvxv.github.io/boke/>

## 本地预览

```bash
hugo server -D
```