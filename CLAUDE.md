# CLAUDE.md

GitHub Pages 上的个人博客（Jekyll）。远端：https://github.com/Jerry2262/docs.git ，线上：https://jerry2262.github.io/docs/ ，git user：shixianjie。

## 仓库结构

- `index.html` — 首页，按日期倒序的卡片列表。
- `_layouts/default.html` — 全站框架（header/footer/css 引入）；`_layouts/post.html` — 文章页，渲染 H1 标题。
- `assets/css/style.css` — 所有样式集中在此一个文件，无外部 CSS。
- `_config.yml` — `defaults` 对所有 posts 自动套 `layout: post`，故文章 front matter 不必写 `layout`；`permalink: /posts/:title/`。
- 无 `_includes/` 目录。

## 偏好

- 文章语言为中文。
- 文件命名：小写 kebab-case 英文描述性文件名。
- 新增文章放 `_posts/`，需 front matter（`title`/`date`/`description`/`category`）；正文不写 H1（标题由 post 布局渲染）。
- 仓库走 main 分支直接开发。
- 提交即推送（用户未特别要求时）。
- GitHub 相关操作优先用 `gh` CLI（已登录账号 `Jerry2262`），不要让用户去网页端。

## 约束

- 站点用自定义布局（`_config.yml` 中 `theme: null`），不要引入外部 Jekyll 主题。
- `baseurl: /docs`，所有内部链接必须用 `relative_url` 过滤器，否则子路径下链接会断。
- `_posts/` 文件名必须为 `YYYY-MM-DD-kebab-title.md`。
