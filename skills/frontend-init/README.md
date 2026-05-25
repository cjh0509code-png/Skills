---
name: 前端项目初始化
emoji: 🚀
category: 开发效率
skill: frontend-init
description: 交互式向导，新建或增强 React/Vue/Svelte/Angular/Solid 项目，含 UI 库、布局模板与工程化配置一键集成。
---

# frontend-init

前端项目脚手架与工程化配置的交互式向导。两种模式：

- **新项目模式** — 主流框架官方脚手架一键创建（Vite / Next.js / Nuxt / SvelteKit / Angular CLI / SolidStart …），按勾选叠加 TS / Tailwind / **UI 组件库** / **项目布局**（后台/官网/Dashboard 等）/ 测试 / lint / git hooks 等
- **增强模式** — 不动现有代码，给已有项目按需补齐：ESLint flat config、Prettier、Husky + lint-staged、commitlint、EditorConfig、CI 模板、严格 TS、UI 组件库等（增强模式不动业务代码，不展示布局选项）

## 功能说明

| 阶段 | 内容 |
|------|------|
| 0 | 模式选择（新建 / 增强） |
| 1 | 新建：选框架 + 包管理器；增强：自动检测现有技术栈 |
| 2 | 工程化配置勾选（**项目布局** / TS / 样式 / **UI 库** / 测试 / lint / hooks / CI） |
| 3 | 命令清单预览，用户确认（dry-run 思路） |
| 4 | 执行：跑官方脚手架 → 装依赖 → 写配置 → 仅 `git init`（不替你 add/commit/push） |
| 5 | 验收：build / lint / test 检查 + Quick Start 指引 |

### 支持的项目布局（按框架 + UI 库三元组过滤）

| 布局 | 说明 | 已沉淀模板 |
|------|------|----------|
| 空白起点 | 仅 `<App />` 占位 | 全框架 |
| 后台管理 (Admin) | Header + 左侧 Sidebar + 内容区 + 可选 Footer，含路由 | React+Antd / React+shadcn / Vue+Element Plus |
| 官网 / Marketing | 顶部导航 + Hero + Features Sections + Footer | React+Antd / React+Tailwind |
| Dashboard 数据看板 | Header + KPI 卡片 Grid + 图表占位区 | React+Antd |
| 文档站 (Docs) | 左侧目录 + 右侧 Markdown + 顶部搜索 | React+Antd（**推荐改用 VitePress/Nextra**） |
| 自定义描述 | 用一句话描述布局，现场生成 | 所有组合通用 |

> 未覆盖的 (框架, UI 库, 布局) 组合 → 自动降级到「空白起点」+ 引导走「自定义描述」。

### 端口策略（避免端口冲突）

- **Webpack 5 模板**：默认 `devServer.port: 'auto'`，自动找可用端口（避免 3000 被占）
- **Vite 模板**：默认 5173，被占自动 +1（Vite 内置）
- **Next.js 模板**：默认 3000，被占自动 +1（Next 内置）
- 想固定端口：`cross-env PORT=4000 npm run dev`（Webpack）或对应框架的 `--port` flag

### 支持的 UI 组件库（按框架动态过滤）

| 框架 | 可选 |
|------|------|
| React | shadcn/ui、Ant Design 5、MUI v6、Mantine v7、Chakra UI v3 |
| Vue | Element Plus、Ant Design Vue 4、Naive UI、Vuetify 3、PrimeVue、shadcn-vue |
| Svelte | shadcn-svelte、Bits UI、Skeleton UI、Flowbite Svelte |
| Angular | Angular Material、PrimeNG、ng-zorro-antd |
| Solid | Solid UI、Kobalte、Hope UI |
| 通用 | daisyUI（Tailwind 插件，跨框架） |

### Git 行为边界

只做 `git init -b main`，初始化后弹一次问询让你**可选**填写：
- 本仓库 `user.name` / `user.email`（跳过则用全局配置）
- 远程 `origin` URL（跳过则不加 remote）

**绝不**替你做 `git add` / `git commit` / `git push`，避免污染你的提交历史。

### 支持的框架与脚手架

| 生态 | 选项 |
|------|------|
| React | Vite + React、Next.js（App Router / Pages）、Remix |
| Vue | Vite + Vue 3、Nuxt 3 |
| Svelte | SvelteKit、Vite + Svelte |
| Angular | Angular CLI（Standalone） |
| Solid | Vite + Solid、SolidStart |
| 纯工程化 | 不绑定框架，只装 ESLint/Prettier/Husky 等 |

### 跨 agent 兼容

Universal 通用版，Claude Code / Cursor / Cline / Gemini Code Assist 均可使用，不依赖 `Task` 等高级特性。

## 触发词

`/frontend-init`、"初始化前端项目"、"新建 React/Vue 项目"、"加 ESLint"、"装 Husky"、"前端工程化"、"补 Prettier 配置"

## 安装

```bash
npx skills add cjh0509code-png/Skills --skill frontend-init
```

## 文件结构

```
frontend-init/
├── README.md
├── SKILL.md                    # 主控向导（状态机）
└── references/
    ├── scaffolds.md              # 各框架官方脚手架命令对照表
    ├── configs.md                # 工程化配置文件模板库
    └── detection.md              # 已有项目技术栈检测策略
```
