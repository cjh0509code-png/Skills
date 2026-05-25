---
name: frontend-init
description: |
  前端项目初始化与工程化配置的交互式向导。两种模式：新建项目（调用 Vite/Next.js/Nuxt/SvelteKit/
  Angular CLI/SolidStart 等官方脚手架）或增强已有项目（补 ESLint/Prettier/Husky/lint-staged/
  commitlint/CI 等工程化配置）。

  务必在以下场景触发，即使用户没明确说"初始化"：
  • "新建一个 React/Vue/Svelte/Angular 项目"、"搭一个前端项目"、"从零开始一个前端"
  • "给项目加 ESLint/Prettier/Husky/commitlint/lint-staged"、"配 git hooks"、"补严格 TS"
  • "前端工程化"、"加 CI"、"做个标准的前端模板"
  • 用户在空目录或新仓库里问"怎么开始一个前端项目"

  不用于：后端项目初始化（Node.js API 不算，除非用户明说前端全栈框架如 Next.js/Nuxt）
  不用于：纯库/SDK 开发的 monorepo 搭建（用专门的 monorepo skill）
  不用于：已有完整工程化的项目改造（如 ESLint v8 → v9 迁移，那是 transformer 类技能的活）
triggers:
  - frontend-init
  - 初始化前端
  - 新建前端项目
  - 前端工程化
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# frontend-init

> 前端项目脚手架 + 工程化配置的交互式向导。
> 跨 agent 通用版：不使用 Task subagent，所有步骤主任务顺序执行。

---

## 角色定位

你是一名资深前端工程师，帮用户：

1. **新项目** — 用官方脚手架（不自创模板）快速搭起一个能跑的项目，按需叠加工程化配置
2. **已有项目** — 不动业务代码，只补齐缺失的工程化设施（lint / format / hooks / CI）

**核心原则**：
- **首选官方脚手架**（`create-vite` / `create-next-app` / `nuxi init` / `create-svelte` / `ng new` 等），不要手动拼 package.json
- **配置文件用最新推荐写法**（ESLint flat config、Prettier 3、Tailwind 4 等）
- **每一步用户都看得见**：先列命令，等确认，再执行
- **执行后必须验证**：跑 `build` / `lint` / `test` 至少一项确认能跑

---

## 状态机（6 阶段，严格推进）

```
[0: 模式]
   ├── 新建 → [1a: 框架+包管理器]
   └── 增强 → [1b: 检测现有栈]
                       ↓
                 [2: 工程化勾选]
                       ↓
                 [3: 命令清单预览 + 用户确认]
                       ↓
                 [4: 执行]
                       ↓
                 [5: 验收 + Quick Start]
```

**禁止**：跳过阶段 3 直接执行；不验收就交付；自创非官方脚手架命令。

---

## 阶段 0：模式选择

> **"你想做哪种？**
>
> **① 新建项目** — 我帮你用官方脚手架（Vite/Next.js/Nuxt/SvelteKit/Angular CLI/SolidStart）创建一个新项目，再按你选的工程化清单一次性装好。
>
> **② 增强已有项目** — 不动你现有代码，给项目补 ESLint / Prettier / Husky / lint-staged / commitlint / CI 模板等工程化配置。
>
> 选哪个？"**

记录 `mode ∈ {new, enhance}`。

**收集工作目录**：
> "再确认下，**项目根目录绝对路径**是？（新建模式下我会在这个目录的父目录创建子文件夹；增强模式下我会直接在这个目录里操作。）"

记录 `${PROJECT_ROOT}`。

---

## 阶段 1a：新建模式 — 框架 + 包管理器

> **"选你要的框架（我会用对应的官方脚手架）：**
>
> **React 生态**
> - `react-vite` — Vite + React（最轻量，纯 SPA）
> - `next-app` — Next.js App Router（推荐，SSR/RSC）
> - `next-pages` — Next.js Pages Router（老项目兼容）
> - `remix` — Remix（Vite 版）
>
> **Vue 生态**
> - `vue-vite` — Vite + Vue 3 + `<script setup>`
> - `nuxt` — Nuxt 3（SSR/SSG）
>
> **Svelte 生态**
> - `sveltekit` — SvelteKit（推荐）
> - `svelte-vite` — Vite + Svelte（纯组件库场景）
>
> **Angular**
> - `angular` — Angular CLI（Standalone components）
>
> **Solid**
> - `solid-vite` — Vite + Solid
> - `solid-start` — SolidStart（SSR）
>
> **其他**
> - `vanilla` — Vite + 纯 TS/JS（不绑定框架）
> - 没看到你要的？告诉我，我查官方脚手架。
>
> **包管理器**：pnpm（推荐）/ npm / yarn / bun？
>
> **项目名**（会作为子文件夹名）？"**

记录：`framework`、`package_manager`、`project_name`。

**Read** `references/scaffolds.md` 拿对应的官方命令模板。

---

## 阶段 1b：增强模式 — 检测现有栈

执行 **检测脚本**（先 Read `references/detection.md` 拿完整算法）：

1. `Read ${PROJECT_ROOT}/package.json` — 提取 `dependencies` + `devDependencies` + `scripts` + `packageManager`
2. `Glob ${PROJECT_ROOT}/{tsconfig,tsconfig.json,jsconfig.json}` — 判断 TS
3. `Glob ${PROJECT_ROOT}/{eslint.config.*,.eslintrc*,prettier.config.*,.prettierrc*,.husky,commitlint.config.*,tailwind.config.*}` — 判断已有工程化
4. `Glob ${PROJECT_ROOT}/.github/workflows/*` — 判断 CI

输出诊断表：

```
🔍 项目检测结果

框架：React 18 (Next.js 14 App Router)
语言：TypeScript（strict: 否）
包管理：pnpm 9.x
样式：Tailwind 3
测试：未配置 ❌

工程化清单：
  ✅ ESLint（next/core-web-vitals）
  ❌ Prettier — 缺失
  ❌ Husky + lint-staged — 缺失
  ❌ commitlint — 缺失
  ❌ EditorConfig — 缺失
  ❌ CI 模板 — 无 GitHub Actions

可选增强：[基于上面 ❌ 的清单]

进入阶段 2 勾选要补哪些？
```

如果检测不出框架（不像前端项目），停下问用户："这看起来不像前端项目（没找到 React/Vue/Svelte/Angular 等依赖），要继续吗？"

---

## 阶段 2：工程化配置勾选

**先 Read `references/configs.md`** 拿所有配置文件模板。

按 `mode` 出菜单：

### 新建模式菜单（默认勾选 ✅ 项）

⚠️ **UI 组件库行必须按 Q1 框架动态过滤**（React 项目不展示 Vue 库），具体库清单见 `references/configs.md` §12。
⚠️ **项目布局类型**按 Q1 框架动态展示对应模板存在与否（见 `references/configs.md` §13），无模板的组合自动只剩"空白起点"+"自定义描述"。

```
请勾选要叠加的工程化配置（用数字组合，如 "1,3,5,7"）：

项目布局类型（单选，决定预生成的初始组件 + 路由结构）：
  0a. ✅ 空白起点          —— 仅 <App /> 占位，不预生成任何业务结构
  0b. □ 后台管理 (Admin)   —— Header + 左侧 Sidebar + 内容区 + 可选 Footer，含路由示例
  0c. □ 官网 / Marketing   —— 顶部导航 + Hero + Features Sections + Footer
  0d. □ Dashboard 数据看板 —— Header + 卡片 Grid + 图表占位区
  0e. □ 文档站 (Docs)      —— 左侧目录 + 右侧 Markdown 内容 + 顶部搜索
  0f. □ 自定义描述         —— 你用一句话告诉我布局，我现场生成

语言与样式：
  1. ✅ TypeScript（strict 模式）
  2. □ Tailwind CSS 4

UI 组件库（按你选的框架动态展示，单选）：
  # React 项目展示：
  3a. □ shadcn/ui          —— 复制式无依赖，Radix + Tailwind，最灵活
  3b. □ Ant Design 5       —— 中后台首选，组件最全
  3c. □ Material-UI v6     —— Google Material 设计
  3d. □ Mantine v7         —— 现代化、Hook 丰富
  3e. □ Chakra UI v3       —— 风格简洁，可访问性好
  3f. □ 不装 / 自己选       —— 跳过

  # Vue 项目展示：
  3a. □ Element Plus       —— 中后台首选
  3b. □ Ant Design Vue 4
  3c. □ Naive UI            —— 轻量、TS 友好
  3d. □ Vuetify 3           —— Material 设计
  3e. □ PrimeVue
  3f. □ shadcn-vue
  3g. □ 不装 / 自己选

  # Svelte 项目展示：
  3a. □ shadcn-svelte
  3b. □ Bits UI（headless）
  3c. □ Skeleton UI
  3d. □ Flowbite Svelte
  3e. □ 不装 / 自己选

  # Angular 项目展示：
  3a. □ Angular Material
  3b. □ PrimeNG
  3c. □ ng-zorro-antd
  3d. □ 不装 / 自己选

  # Solid 项目展示：
  3a. □ Solid UI（shadcn-style）
  3b. □ Kobalte（headless）
  3c. □ Hope UI
  3d. □ 不装 / 自己选

  # vanilla / 选 Tailwind 时通用追加：
  3z. □ daisyUI            —— Tailwind 插件，跨框架可用

代码质量：
  4. ✅ ESLint（flat config，含框架专用规则）
  5. ✅ Prettier 3（含 Tailwind 排序插件，如选了 Tailwind）

测试：
  6. □ Vitest（单元 + 组件）
  7. □ Playwright（E2E）

Git Hooks：
  8. □ Husky + lint-staged（commit 前自动 lint+format）
  9. □ commitlint（强制 Conventional Commits）

杂项：
  10. ✅ EditorConfig + .gitignore（标准模板）
  11. □ GitHub Actions CI（PR 时跑 lint/build/test）
  12. □ README 模板（项目名 + 安装/开发/构建脚本）

回我数字组合，或输入 "默认" 用 ✅ 项。
```

**UI 库选中后的安装路径**：去 `references/configs.md` §12 查对应章节，拿安装命令 + 配置文件模板 + 入口注入代码（如 React 的 `App.tsx` 加 Provider）。

**布局类型选中后的生成路径**：去 `references/configs.md` §13 查对应章节，拿 `src/App.tsx`（或 `App.vue` / `+layout.svelte`）+ 必要的路由依赖（如 React 的 `react-router-dom`、Vue 的 `vue-router`）+ 入口注入代码。**布局必须与 UI 库联动**：选了 Antd 的"后台管理"应该用 `antd Layout/Menu` 而不是手写 div；选了 shadcn 的"后台管理"应该用 shadcn 的 `Sidebar` 组件——具体模板按 `(框架, UI库, 布局)` 三元组查 §13 内的小节。

### 增强模式菜单

只显示阶段 1b 检测到 ❌ 的项，用户勾选要补哪些。增强模式**也可以**追加 UI 组件库（按已检测到的框架展示对应可选项）。**增强模式不动业务代码**，所以不展示布局选项。

---

## 阶段 3：命令清单预览 + 用户确认（必经）

汇总所有要执行的命令 + 要写的文件，**完整列出**给用户看：

```
📋 接下来会执行：

【脚手架】（新建模式才有）
$ cd <project_root_parent>
$ pnpm create vite@latest my-app --template react-ts
$ cd my-app

【依赖安装】
$ pnpm add -D eslint @eslint/js typescript-eslint
$ pnpm add -D prettier prettier-plugin-tailwindcss
$ pnpm add -D husky lint-staged
$ pnpm add -D @commitlint/cli @commitlint/config-conventional

【配置文件】
✏️  eslint.config.js     （新建）
✏️  .prettierrc.json     （新建）
✏️  .editorconfig        （新建）
✏️  .husky/pre-commit    （新建）
✏️  .husky/commit-msg    （新建）
✏️  commitlint.config.js （新建）
✏️  .github/workflows/ci.yml （新建）
✏️  package.json         （加 scripts: lint/format/prepare + lint-staged 配置）

【Git】（只 init，不替你 add / commit / push）
$ git init -b main（如未初始化）
↓
完成后会弹一次问询，全部可选可跳过：
  • git user.name / user.email（跳过则用全局/系统配置）
  • 远程仓库 origin URL（跳过则不加远程，你后续自己 `git remote add origin <url>`）
首次 `git add` / `git commit` / `git push` 都由你自己来。

⚠️  全部命令在 ${PROJECT_ROOT} 下执行。

确认执行？输入 "go" 开始，或告诉我要改/删哪些。
```

**必须等用户明确确认**才进入阶段 4。用户说要改 → 回到阶段 2 调整。

---

## 阶段 4：执行

按上面的清单顺序执行。每一步都 `Bash` 跑命令并查 exit code。

### 4.1 脚手架（新建模式）

从 `references/scaffolds.md` 取对应命令，调用 `Bash` 执行。

**关键**：所有官方脚手架都用 `--yes` / 非交互参数（如 `--template`），避免卡在交互式提问上。具体参数见 `references/scaffolds.md`。

如命令失败：读取 stderr，给用户报错，问是否重试/换包管理器/手动处理。

### 4.2 依赖安装

按包管理器拼命令：
- pnpm: `pnpm add -D <pkgs>`
- npm: `npm install -D <pkgs>`
- yarn: `yarn add -D <pkgs>`
- bun: `bun add -d <pkgs>`

合并同类调用（一次装多个 -D），减少 IO。

### 4.3 写配置文件

用 **Write 工具** 写每个文件（不用 cat heredoc）。文件内容从 `references/configs.md` 取模板，按用户的技术栈做小幅替换（如 ESLint 规则插件根据框架不同）。

**冲突处理**：写文件前用 `Read` 探测目标文件是否存在：
- 不存在 → 直接 Write
- 已存在且是新建模式（脚手架刚生成的） → 用 Read + Edit 合并
- 已存在且是增强模式 → 停下问用户："`<path>` 已存在，要 [覆盖 / 合并 / 跳过]？"

#### 4.3.x 布局生成的目录约束（**重要**）

如果用户在阶段 2 选了**非空白**布局（后台 / 官网 / Dashboard / 文档站），**必须**按 `references/configs.md §13.x` 的多文件目录结构生成，**不要**把 layout/pages/router 全堆在单文件 `App.tsx` 里。

最小目录约束（按布局类型动态）：

| 布局类型 | 必须生成的目录 |
|---------|-------------|
| 后台管理 | `src/layouts/`、`src/pages/`、`src/router/`、`src/components/`（空 + `.gitkeep`） |
| 官网 / Marketing | `src/sections/`（Hero/Features/Footer 等切片）、`src/components/` |
| Dashboard 数据看板 | `src/widgets/`（KPI 卡片、图表组件等）、`src/pages/`（如只有一页可省 router） |
| 文档站 | 不要自拼，**默认建议改用 VitePress** |
| 空白 | 不强制目录 |

#### 4.3.y 必跑 prettier --write src/（写完所有源码后）

**Tailwind 4 + prettier-plugin-tailwindcss 对类顺序有严格要求**，但 skill 生成的代码类顺序不一定符合，每次都让 `format:check` 失败再手工补很 dumb。
**写完所有 `src/**/*` 文件后立即跑一次**：

```bash
cd "${PROJECT_ROOT}/${project_name}"
npx prettier --write src/
```

避免阶段 5 验收 `format:check` 卡红色。

### 4.4 改 package.json scripts

读 → 用 Edit 合并以下 scripts（如不存在）：

```json
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "prepare": "husky"
  }
}
```

如选了 Husky，加 `"lint-staged"` 配置块。

### 4.5 初始化 Husky（如选了）

```bash
pnpm exec husky init
# 写 .husky/pre-commit、.husky/commit-msg
```

### 4.6 Git init（只 init + 可选问询，**绝不**替用户 add/commit/push）

**核心约束**：本 skill 的 Git 边界**仅限 `git init`**。`git add` / `git commit` / `git remote add` / `git push` 全部由用户自己来——避免把用户不想要的初始 commit 强行塞进历史，污染 blame / signed commits 流程。

#### 步骤 1：仅做 `git init`（如目录无 `.git`）

```bash
git init -b main   # 显式指定 main 避免不同环境默认分支不一致
```

如已有 `.git`，**跳过整个 4.6**，直接告诉用户："已检测到 git 仓库，跳过 init"。

#### 步骤 2：弹一次问询（全部可选）

用 `AskUserQuestion` 工具弹一个**多问题**问询，让用户**可选**填写：

> **"Git 已初始化完成。下面这些是可选项，跳过的话由你自己后续配置：**
>
> 1. **本仓库 `user.name`**（用于 commit author，留空则用全局 `~/.gitconfig`）
> 2. **本仓库 `user.email`**（同上）
> 3. **远程仓库 origin URL**（留空则不加 remote，你之后可以自己 `git remote add origin <url>`）
>
> 想全跳过直接回 'skip'。"

#### 步骤 3：按用户回答执行（每项独立可选）

```bash
# 仅当用户填了 user.name：
git -C "${PROJECT_ROOT}" config user.name "<填的值>"

# 仅当用户填了 user.email：
git -C "${PROJECT_ROOT}" config user.email "<填的值>"

# 仅当用户填了远程 URL：
git -C "${PROJECT_ROOT}" remote add origin "<填的 URL>"
```

#### 步骤 4：明确告知用户后续动作

执行完上述配置后，**必须**在阶段 5 Quick Start 里告诉用户：

> "📌 Git 仓库已 init（分支 `main`），但**首次提交需要你自己来**：
> ```bash
> cd <project>
> git add .
> git commit -m '<你想写的消息>'
> # 如已配 origin：
> git push -u origin main
> ```"

**绝对禁止**：
- ❌ `git add .`（不替用户 stage）
- ❌ `git commit -m "..."`（不替用户做首次提交，签名/作者/消息都该是用户的选择）
- ❌ `git push`（任何 push 都禁止）
- ❌ 调用 `gh repo create` 或类似动作（不替用户在远程平台创建仓库）

---

## 阶段 5：验收 + Quick Start

按选的工程化清单跑验证命令：

| 选了什么 | 跑什么验证 |
|---------|-----------|
| 任何框架 | `<pm> run build` — 确保能构建（新建模式必跑） |
| ESLint | `<pm> run lint` |
| Prettier | `<pm> run format:check` |
| Vitest | `<pm> exec vitest run` |
| Husky | `ls -la .husky/` 确认钩子文件存在且可执行 |
| commitlint（有 commit-msg hook） | 真触发测试：`echo "bad" \| npx --no -- commitlint > /dev/null 2>&1 ; echo $?` 应输出非 0；`echo "feat: x" \| npx --no -- commitlint > /dev/null 2>&1 ; echo $?` 应输出 0 |
| **布局类型≠空白**（后台/官网/Dashboard 等）| **运行时人工 check（写入 Quick Start）**：起 `<pm> dev` → 浏览器打开实际端口 → 确认 ① 布局**撑满全屏无四周白边** ② 内容区高度正确 ③ 菜单/header 等 UI 库组件样式没被 Tailwind preflight 清掉。**这个 check 不能用 build 覆盖**，必须写进 Quick Start 提醒用户人肉看一眼 |

> ⚠️ **dev server 端口**不在自动验收范围（`build` 已能覆盖编译正确性），原因：dev 端口是运行时行为，且本 skill 模板默认用 `port: 'auto'`（webpack-dev-server）/ 框架默认值（Vite/Next 自带冲突回避）。Quick Start 里会明确告诉用户「实际端口看 dev 启动输出」。

> ⚠️ **布局/样式 bug 是 silent failure**：`build` 通过 ≠ 视觉正确。常见两个坑：
> 1. 跳过 Tailwind preflight 后没补 `html/body/#root { height: 100% }` reset → 后台布局四周白边（见 configs.md §12.8）
> 2. AdminLayout 用 `min-h-screen` 而非 `h-full` → 某些 viewport 内容区高度塌陷（见 configs.md §13.1.1）
>
> 这两个都会被 build 静默放过，所以 **Quick Start 必须提示用户人肉打开浏览器看一眼**。

**单条失败**：报告失败原因，给修复建议（如 lint 报错的话直接 `lint:fix`），不阻断 Quick Start 输出。

**全部跑完后**输出 Quick Start：

```
✅ 项目初始化完成！

📁 项目位置：${PROJECT_ROOT}/<project_name>

🚀 开始开发：
  cd <project_name>
  <pm> dev
  # ⚠️ 实际端口看启动输出 —— webpack 模板用 port: 'auto'，
  # 想固定端口跑：cross-env PORT=4000 <pm> dev（webpack）
  # 或直接改 webpack.config.ts 里 devServer.port

  # 🔴 必看：起 dev 后立刻打开浏览器人肉确认（build 不能验视觉）：
  #   ① 布局是否撑满全屏，四周无白边
  #   ② 内容区高度正确（不该塌陷）
  #   ③ UI 库组件（Antd Button/Menu 等）默认样式正常
  # 如发现白边：检查 src/index.css 是否含跳 preflight 后的最小 reset
  # 如发现高度塌陷：layout 容器用 h-full 不是 min-h-screen

🔧 常用脚本：
  <pm> run dev          # 开发服务器
  <pm> run build        # 生产构建
  <pm> run lint         # 代码检查
  <pm> run lint:fix     # 自动修复
  <pm> run format       # 格式化全部文件
  <pm> exec vitest      # 单元测试（如选了 Vitest）

📋 工程化设施：
  [按勾选清单列出已配置的项 + 简短一句话说明每个的作用]

📌 Git 状态：
  • 仓库已 init（分支 main）
  • [如用户在阶段 4.6 配了 user.name/email，列出来]
  • [如用户配了 origin，列出 URL；没配则提示 "未配远程，需自己 git remote add origin <url>"]
  • ⚠️ 尚未做首次提交，由你自己来：
      git add .
      git commit -m '<message>'
      # 若已配 origin：
      git push -u origin main

📝 后续建议：
  • CI 已就绪（如选了）：推到 GitHub 后自动跑 lint/build/test
  • Commit 规范（如选了 commitlint）：feat: / fix: / docs: / chore: / refactor: ...
  • UI 库（如选了）：组件用法见对应官方文档链接
```

---

## 错误恢复

| 情况 | 处理 |
|------|------|
| 用户跑到一半改主意 | 回到对应阶段重选，已写的文件可选保留/回滚（`git status` + 询问） |
| 网络问题导致 `create-*` 失败 | 报错给用户，建议换 registry（`npm config set registry https://registry.npmmirror.com`）或重试 |
| 包管理器未安装（如选了 pnpm 但没装） | 自动降级到 npm + 提示用户："未检测到 pnpm，已用 npm；建议 `npm i -g pnpm` 后续用" |
| 目标目录非空（新建模式） | 停下问："`<dir>` 已存在文件，要 [换名 / 强制 / 取消]？" |
| 已有 ESLint v8 配置（增强模式） | 不自动升级 v9，告知用户"已有 .eslintrc，要升级到 flat config 需单独操作（不在本 skill 范围）" |
| Windows 路径含中文/空格 | 所有 Bash 调用必须把路径用双引号包裹；Node 脚本优先用 `path.join` |

---

## 跨平台注意

- **路径**：始终用绝对路径 + 双引号；不要用 `~`（Windows 不展开）
- **Shell**：所有 Bash 命令必须能在 Git Bash 上跑。禁止用 `&&` 之外的复杂管道（如 `<(...)`）
- **包管理器命令**：bun 的 dev dep 是 `-d`（小写），其他都是 `-D`（大写），注意区分
- **husky init**：v9+ 不再用 `.husky/_/husky.sh` 引导，直接写脚本即可

---

## 与 references/ 的协作约定

| references 文件 | 何时读取 |
|----------------|---------|
| `scaffolds.md` | 阶段 1a（拿官方脚手架命令）+ 阶段 4.1（执行时再确认参数） |
| `configs.md` | 阶段 2（菜单展示）+ 阶段 4.3（取文件模板） |
| `detection.md` | 阶段 1b（检测已有项目栈） |

**重要**：按需读取，不要在阶段 0 就把所有 references 加载到 context。
