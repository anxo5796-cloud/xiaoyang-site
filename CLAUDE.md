# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Navfolio 主站：一个基于 Astro 的"安静个人发布空间"starter，同时是 Navfolio 生态（`@navfolio/*` 包）的组合根。当前产品分支 `v1`。仓库工作语言是中文。

维护者写的 agent 指南在 `AGENT.md`；当前架构边界与能力落地状态在 `.agents/context/current-design.md` 与 `.agents/context/current-progress.md`，架构变化时要同步更新。

## 常用命令

```bash
bun install              # 安装依赖（GitHub spec 依赖由 bun.lock 锁定）
bun run dev              # 开发服务器
bun run build            # fonts:ui（字体子集）→ astro build → pagefind 搜索索引
bun run preview          # 预览生产构建
bun test                 # 运行单元测试（*.test.ts，bun 自带 runner，无 npm test）
bun run lint             # eslint
bun run format           # prettier 写回
bun run format:check     # lint + prettier 检查（lint-staged 对每次提交执行）
bun run post:new my-post          # 新建博客内容；project:new / vibe:new / media:new 同理
bun run post:new my-post --mdx    # 可选 --md/--mdx 与末尾输出目录参数
bun run fonts:ui         # 单独验证字体子集化环境
```

验证顺序（AGENT.md）：先 `bun run format:check`，再按影响范围 `bun run build`。可见 UI、路由、导航、样式或 hydration 改动还要浏览器检查。

### 构建硬依赖

`bun run build` 硬依赖 Python 3 + fonttools + brotli，用于生成中日韩字体的 WOFF2 子集；脚本自动优先使用项目 `.venv`（不要求激活）。缺这些工具构建会直接失败。Node 要求 >= 22.12（`.nvmrc` 为 22.12.0），脚本运行时用 Bun。

## 架构

组合流程：

```
navfolio.config.ts（显式注册 projects/vibe/media 模块 + markdown/pages 插件）
  → src/plugins/config.ts（模块解析、Astro 集成与 remark/rehype 聚合）
  → astro.config.mjs（integrations、注入路由、Tailwind、virtual alias）
```

- **依赖方向**：`@navfolio/core`、`@navfolio/pages`、`@navfolio/theme-default`、`@navfolio/plugin-markdown`、`@navfolio/mdx-components` 都通过 GitHub spec 安装并由 lockfile 锁定。行为应改在真正的 owner 仓库，不要因为主站是组合根就把逻辑写回主站；上游改动需先推送再刷新本仓库 lockfile。
- **内容 schema**：`src/content.config.ts` 集中拥有全部 collection 的 Zod schema，并按 `navfolio.config.ts` 的模块启用状态条件注册 projects/vibe/media。关闭模块后对应 route、collection、导航、scaffold、i18n 贡献都应消失。
- **站点配置**：`src/config/site.toml` 是用户可编辑配置面（身份、色板、字体、页面文案、导航、搜索、评论、首页卡片），由 `src/content.config.ts` 的 schema 校验，字段缺失或类型错误时构建直接报错。读取入口是 `src/data/site.ts` 的 `getSiteConfig()`。
- **页面模块**：Projects 的 index/detail UI 仍在 `src/modules/routes/**`；Vibe 与 Media 使用 package-owned routes，通过 `virtual:navfolio/page-runtime`（alias 指向 `src/modules/page-runtime.ts`）访问主站组件与 helper，不得直接 import 主站私有路径。
- **i18n**：`src/i18n/{en,zh-CN,zh-TW}.json` 经 `src/utils/ui-text.ts` 的 `createI18n`（来自 `@navfolio/core`）加载；页面文案不硬编码。
- **内容**：`src/content/{blog,projects,vibe,media}` 与 `about.mdx`。新建内容走脚手架脚本（`scripts/new-content.ts`），模板在 `scripts/templates/post.md` 与各 page package 发布的 `templates/default.md`。
- **部署**：GitHub Pages workflow（push `v1` 触发）；`SITE_URL`/`SITE_BASE` 环境变量可覆盖站点 url/base；构建产物 `dist/` + `dist/pagefind`。保持静态输出、无 secret 入库、GitHub Pages 子路径兼容。
- **设计约束**：保持 calm editorial 视觉、可访问性、响应式行为和无 JavaScript 的基本可读性。

## 当前状态注意（2026-08-02）

docs 内容模式已过期：git 已移除 `src/docs` submodule（commit 5d10e5c、33b717e），但 `AGENT.md`、`.github/workflows/deploy-pages.yml`、`vercel.json` 仍引用它；`bun run docs:build` / `docs:dev`（`NAVFOLIO_CONTENT_SOURCE=docs`）目前不可用，CI 的 submodule 更新是空操作。
