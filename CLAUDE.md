# CLAUDE.md

本仓库是一个部署在 GitHub Pages 上的个人博客，记录计算体系、投资逻辑与日常思考类文章。

## 仓库信息

- 远端：https://github.com/Jerry2262/docs.git
- Pages 地址：https://jerry2262.github.io/docs/
- 作者 / git user：shixianjie

## 技术栈

- 静态生成：Jekyll（GitHub Pages 原生构建，无需 CI）
- 配置：`_config.yml` 中 `theme: null`，使用自定义布局
- `baseurl: /docs`（项目站点，所有内部链接需用 `relative_url` 过滤器）
- 时区 `Asia/Shanghai`，permalink `/posts/:title/`

## 目录结构

- `index.html` — 主页：渐变标题 hero + 响应式卡片网格，自动按日期倒序列出 `site.posts`
- `_layouts/default.html` — 全站框架：sticky 毛玻璃顶栏 + 页脚
- `_layouts/post.html` — 文章页布局
- `assets/css/style.css` — 样式（现代卡片风，自带暗色模式）
- `_posts/` — 文章源文件，文件名必须为 `YYYY-MM-DD-kebab-title.md`

## 新增文章

在 `_posts/` 新建 `YYYY-MM-DD-标题.md`，顶部 front matter：

```yaml
---
layout: post
title: "标题"
date: YYYY-MM-DD
description: "一句话摘要，显示在主页卡片上"
category: 分类
---
```

正文不要写 H1（标题由布局渲染）；从 `##` 开始。推送到 main 即自动部署。

## 约定与偏好

- 文件命名：小写 kebab-case 英文描述性文件名（如 `simd-simt-parallel-computing-semantics.md`）。
- 文章语言为中文。
- 提交后若需同步到远端：`git push origin main`（用户未特别要求时，提交即推送）。
- 仓库走 main 分支直接开发。

## 本地预览

本机未安装 Jekyll。如需本地预览：

```bash
gem install jekyll
bundle exec jekyll serve --baseurl /docs
# 或
jekyll serve --baseurl /docs
```

## GitHub Pages 开启方式

Settings → Pages → Source: Deploy from a branch → Branch: `main` /(root)。已开启一次后无需重复设置。
