# 已有项目技术栈检测算法

阶段 1b（增强模式）用本算法识别项目现状，输出诊断表。

---

## 检测步骤

### Step 1：识别项目存在性

`Read ${PROJECT_ROOT}/package.json`：

- 失败 → 报错："该目录没有 package.json，不像前端项目。要在这个目录运行 `<pm> init` 创建一个空 package.json 吗？"
- 成功 → 继续

### Step 2：识别框架

按以下优先级查 `dependencies` + `devDependencies`：

| 检测键 | 判定框架 | 子版本判定 |
|--------|---------|-----------|
| `next` | Next.js | `next@>=13` + 存在 `app/` 目录 → App Router；否则 Pages |
| `nuxt` | Nuxt | `nuxt@>=3` → Nuxt 3 |
| `@sveltejs/kit` | SvelteKit | — |
| `remix` 或 `@remix-run/*` | Remix | — |
| `@angular/core` | Angular | 看 `@angular/core` 版本 |
| `solid-start` 或 `@solidjs/start` | SolidStart | — |
| `react` | React (无 SSR 框架) | 看是否有 `vite` → Vite + React |
| `vue` | Vue (无 SSR 框架) | 看版本 `vue@>=3` → Vue 3 |
| `svelte`（无 kit） | 纯 Svelte | 看是否有 `vite` |
| `solid-js`（无 start） | 纯 Solid | — |
| 无任何前端框架 | "非前端项目，是否继续？" | — |

### Step 3：识别语言

- 有 `tsconfig.json` → TypeScript
- 检查 `tsconfig.json` 的 `compilerOptions.strict` → 判断是否 strict
- 无 tsconfig 但有 `jsconfig.json` → JS（带路径别名）
- 都没有 → JS

### Step 4：识别包管理器

按优先级：

1. `package.json` 的 `"packageManager"` 字段（最权威）
2. 锁文件存在性：
   - `pnpm-lock.yaml` → pnpm
   - `yarn.lock` → yarn
   - `bun.lockb` 或 `bun.lock` → bun
   - `package-lock.json` → npm
3. 都没有 → 默认 npm

### Step 5：识别样式方案

| 检测 | 判定 |
|------|------|
| 有 `tailwindcss` 依赖 | Tailwind（看版本判 3/4） |
| 有 `unocss` 依赖 | UnoCSS |
| 有 `sass` / `node-sass` | Sass |
| 有 `styled-components` | styled-components |
| 有 `@emotion/*` | Emotion |
| 有 `*.module.css` 文件（用 Glob 查） | CSS Modules |
| 都没有 | 普通 CSS |

### Step 6：识别测试

| 检测 | 判定 |
|------|------|
| 有 `vitest` | Vitest |
| 有 `jest` | Jest |
| 有 `@playwright/test` | Playwright |
| 有 `cypress` | Cypress |
| `package.json scripts.test` 存在但无上述任何一个 | 自定义 test，标 "已有但未识别" |

### Step 7：识别工程化设施（核心，决定阶段 2 菜单）

对每项判断「已配置 ✅」/「缺失 ❌」：

| 项目 | 检测方法 |
|------|---------|
| ESLint | `eslint.config.{js,mjs,ts}` 或 `.eslintrc*` 存在；或 `dependencies` 含 `eslint` |
| Prettier | `prettier.config.{js,mjs,cjs}` 或 `.prettierrc*` 存在；或 `dependencies` 含 `prettier` |
| Husky | `.husky/` 目录存在 且 `package.json scripts.prepare` 含 `husky` |
| lint-staged | `package.json` 含 `lint-staged` 字段 或 `lint-staged.config.*` 存在 |
| commitlint | `commitlint.config.{js,cjs,mjs}` 或 `.commitlintrc*` 存在 |
| EditorConfig | `.editorconfig` 存在 |
| CI | `.github/workflows/*.yml` 存在（任一） |
| Tailwind | 见 Step 5 |
| Strict TS | tsconfig 的 `compilerOptions.strict === true` |

### Step 8：识别 git 状态

```bash
git -C "${PROJECT_ROOT}" rev-parse --is-inside-work-tree 2>/dev/null
```

- exit 0 → 已是 git 仓库
- 否则 → 未初始化

---

## 输出格式

把检测结果汇总为下面这种诊断表（阶段 1b 末尾展示给用户）：

```
🔍 项目检测结果

  框架：<framework_name>（<version + 子版本，如 "Next.js 14, App Router"》）
  语言：<TS strict | TS | JS>
  包管理：<pm + version>
  样式：<style_scheme>
  测试：<test_framework | 未配置 ❌>
  Git：<已初始化 | 未初始化>

工程化清单：
  <为每项打 ✅ / ❌>
  ✅ ESLint
  ❌ Prettier — 缺失
  ❌ Husky + lint-staged — 缺失
  ❌ commitlint — 缺失
  ❌ EditorConfig — 缺失
  ❌ CI（无 .github/workflows） — 缺失

可选增强（基于上面 ❌）：
  1. Prettier 3
  2. Husky v9 + lint-staged
  3. commitlint（Conventional Commits）
  4. EditorConfig
  5. GitHub Actions CI 模板
  6. TypeScript strict（如未开）

进入下一步勾选要补哪些？
```

---

## 边界处理

| 情况 | 处理 |
|------|------|
| package.json 解析失败（JSON 损坏） | 报具体错（哪行），让用户修复后重试 |
| monorepo（含 `workspaces` 字段或 `pnpm-workspace.yaml`） | 提示："检测到 monorepo，本 skill 只增强根目录配置，子包需单独运行" |
| 框架版本明显过旧（如 Vue 2、Angular < 14） | 标注 "⚠️ 框架版本较旧"，但仍允许加 lint/prettier 等通用工具 |
| 已有 ESLint v8 (.eslintrc)，无 flat config | 提示："本 skill 不自动迁移 ESLint v8 → v9 flat config"，可继续在旧配置上加 Prettier 等其他设施 |
| 多框架混合（如同时有 react + vue 依赖） | 报错："检测到多个框架，需用户明确告诉我以哪个为主" |
