# AGENTS.md

## Commands

```bash
pnpm install          # required package manager (pnpm-lock.yaml)
pnpm run dev          # start dev server
pnpm run build        # build to .vitepress/dist
pnpm run preview      # preview built output
pnpm exec prettier --write .   # format (no lint/typecheck/test scripts exist)
```

No tsconfig.json, no ESLint, no tests. Less is used for styles, not SCSS.

## Architecture

This is a **VitePress 1.x custom theme** — not a standalone app. The theme entrypoint is `.vitepress/theme/index.ts` (exports `Layout` + `enhanceApp`).

### Key files

| Path | Role |
|---|---|
| `.vitepress/config.mts` | VitePress config, `ThemeConfig` interface, markdown plugins |
| `.vitepress/theme/Layout.vue` | Root layout, routes by `page.filePath` (`index.md` / `tags/index.md` / post) |
| `.vitepress/theme/utils/posts.data.mts` | VitePress data loader — reads `posts/*.md`, parses frontmatter with `gray-matter`, populates `data` export |
| `.vitepress/theme/store/index.ts` | Vue 3 reactive state (selectedPosts, currTag, currPage, splashLoading, fireworksEnabled, etc.) |

### Data flow

- Posts live as `.md` files in `posts/`. Frontmatter fields: `title`, `date`, `tags`, `pinned`, `cover`, `excerpt` (before `---` separator in body).
- `posts.data.mts` loads all posts at build time, sorts pinned-first, then by date descending. The `data` export is consumed by Vue components directly.
- Pagination is client-side with URL query params (`?page=N`, `?tag=X`). The store drives current page/tag state.

### Styling

- **Less** (not SCSS), with scoped styles in Vue SFCs.
- Theme colors are CSS custom properties defined in `.vitepress/theme/styles/vars.less`.
- **Dark mode** uses `html[theme='dark']` selector (not `:root[theme='dark']` or class-based). The `darkMode` state in the store tracks `'light' | 'dark' | 'system'`.
- Background: banner images go in `.vitepress/theme/assets/banner/` (`banner.webp` / `banner_dark.webp`). Background music at same path as `bgm.mp3`.

### Markdown

- Custom plugin: `markdown-it-custom-attrs` adds `data-fancybox="gallery"` to all images (Fancybox lightbox).
- Math: `markdown-it-mathjax3` enabled via `math: true` in config.
- Code highlighting: `solarized-dark` theme, line numbers on.

### Template repo

This repo is designed to be used via "Use this template" on GitHub. Users customize:
- `.vitepress/config.mts` — site title, social links, Gitalk credentials, etc.
- `.vitepress/theme/assets/banner/` — banner images and bgm
- `posts/` — blog content

## Gotchas

- The `ignoreDeadLinks: true` config is set — dead internal links won't break builds.
- The Splash screen blocks scroll during animation; uses `animejs` for fade-out after 1.2–1.5s random delay.
- Spine character player is wrapped in `<ClientOnly>` (SSR-incompatible).
- The `ConvertWebp.sh` script is a todo/helper, not part of the build pipeline.
- Package `type: "module"` — all `.mts`/`.mjs` files use ESM. `__dirname` is polyfilled via `fileURLToPath`.
- `Blueaka` / `Blueaka_Bold` are custom fonts loaded from `public/font/`.

## 微架构安全系列的编写目标

- 能沿同一个例子讲通的部件和过程放在同一篇，用小标题讲透各自的原理、推导和边界。需要独立的前提和例题，才另成一节。
- 篇名用读者能直接理解的说法，点出文章具体解释的事情。必要的术语可以放在篇名、摘要或标签里。导读和前后篇使用与文章一致的标题。
- 导读、先修链接和前后篇链接负责把各节串起来。读者可连续学习，也可从相关专题直接进入所需的一节。
- 新系列用一篇普通文章作为总目录。目录文章在 frontmatter 声明 `bookTitle`，并用 `## 目录 {#book-contents}` 开始分章节列出课程。各正文的 `book` 填目录文章的文件名主体，顶部导航的「系列」页面据此展示入口，正文篇末提供返回目录的入口。
- 正文在博客内讲完。把来自手册、论文或源码的关键规定、过程和含义用中文讲清，必要的先修知识链接到站内文章。站外的论文、文档、源码和配图出处集中放在篇末「参考文献」中，供读者核对。
- 硬件篇先交代整体结构。需要配图时，先查官方文档、开源项目和论文里与正文匹配的原图。核对版本、复用许可和可读性，在正文用中文解释相关结构，图源放在篇末。字段、数值或事件顺序用表格或代码更容易说清时，直接用表格或代码。确实需要看模块与信号的连接、又找不到合用原图时，再自绘框图。自绘采用白底黑线和浅灰分区，统一字体、对齐与线宽，并在窄屏下检查字号、间距和连线。
- 结论依赖代码时给出足以独立阅读的完整示例，逐步对应代码、初态、关键数据和输出。反汇编注明来源与编译条件。能帮助理解的时序或结构图采用正文同一组地址、变量和事件编号，自绘图标明是示例还是实测。
