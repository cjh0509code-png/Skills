# 官方脚手架命令对照表

各主流前端框架的**官方**初始化命令。所有命令用非交互参数，避免卡在 prompt。

> ⚠️ 包管理器变量：以下 `<pm-create>` 替换为：
> - pnpm → `pnpm create`
> - npm → `npm create`
> - yarn → `yarn create`
> - bun → `bun create`

---

## React 生态

### `react-vite` — Vite + React

```bash
<pm-create> vite@latest <project-name> -- --template react-ts
# JS 版：--template react
cd <project-name>
<pm> install
```

### `next-app` — Next.js App Router

```bash
<pm-create> next-app@latest <project-name> \
  --ts --eslint --tailwind --app --src-dir --import-alias "@/*" \
  --use-<pm>
# 各 flag 含义：
#   --ts            TypeScript
#   --eslint        装 Next 默认 ESLint
#   --tailwind      装 Tailwind
#   --app           App Router（推荐）
#   --src-dir       使用 src/ 目录
#   --import-alias  路径别名
#   --use-pnpm/npm/yarn/bun
```

### `next-pages` — Next.js Pages Router

把 `--app` 换成 `--no-app`，其余同上。

### `remix` — Remix（Vite 版）

```bash
<pm-create> remix@latest <project-name> --yes --no-install --no-git-init
cd <project-name>
<pm> install
```
（Remix 2 默认 Vite，不需要额外 flag）

---

## Vue 生态

### `vue-vite` — Vite + Vue 3

```bash
<pm-create> vue@latest <project-name> -- \
  --ts --router --pinia --eslint --prettier --vitest
# 可选 flag：
#   --jsx                启用 JSX
#   --cypress            E2E（与 --playwright 二选一）
#   --playwright         E2E
#   --eslint-with-prettier  ESLint 集成 Prettier
```

### `nuxt` — Nuxt 3

```bash
<pm> dlx nuxi@latest init <project-name> --packageManager <pm> --gitInit --no-install
cd <project-name>
<pm> install
```

> `pnpm dlx` / `npx` / `yarn dlx` / `bunx` 对应各包管理器。

---

## Svelte 生态

### `sveltekit` — SvelteKit

```bash
<pm-create> svelte@latest <project-name>
# 注：create-svelte 暂无完整非交互 flag，阶段 4 执行时给用户预提示
# 关键交互项：template (skeleton/demo)、TS (Yes/JSDoc/No)、ESLint/Prettier/Playwright/Vitest
```

> ⚠️ SvelteKit 脚手架是交互式的，无法 `--yes` 一把过。阶段 3 命令预览时**显式告诉用户**会有 3-5 个交互式选项要回答，建议选：
> - Template: `skeleton` （干净起点）
> - TypeScript: `Yes, using TypeScript syntax`
> - 其余按用户阶段 2 勾选项映射

### `svelte-vite` — Vite + Svelte（无 SvelteKit）

```bash
<pm-create> vite@latest <project-name> -- --template svelte-ts
```

---

## Angular

### `angular` — Angular CLI

```bash
<pm> dlx -p @angular/cli@latest ng new <project-name> \
  --routing --style=scss --strict --standalone \
  --skip-git --package-manager=<pm>
# flag：
#   --routing       带路由
#   --style         样式预处理器（css/scss/sass/less）
#   --strict        严格模式（默认 true，显式声明更稳）
#   --standalone    standalone components（推荐）
#   --skip-git      跳过 git init（由本 skill 阶段 4.6 统一处理）
```

---

## Solid 生态

### `solid-vite` — Vite + Solid

```bash
<pm-create> vite@latest <project-name> -- --template solid-ts
# JS 版：solid
```

### `solid-start` — SolidStart

```bash
<pm-create> solid@latest <project-name>
# 同 SvelteKit，交互式向导
# 关键交互：template (basic/with-tailwindcss/with-mdx/...)、TS、SSR
```

---

## 其他

### `vanilla` — Vite + 纯 TS

```bash
<pm-create> vite@latest <project-name> -- --template vanilla-ts
# JS 版：vanilla
```

---

## 包管理器统一映射

| 操作 | pnpm | npm | yarn | bun |
|------|------|-----|------|-----|
| 创建（create-*） | `pnpm create` | `npm create` | `yarn create` | `bun create` |
| 一次性执行（dlx） | `pnpm dlx` | `npx` | `yarn dlx` | `bunx` |
| 安装依赖 | `pnpm install` | `npm install` | `yarn install` | `bun install` |
| 加 dev 依赖 | `pnpm add -D` | `npm install -D` | `yarn add -D` | `bun add -d` ⚠️ |
| 加生产依赖 | `pnpm add` | `npm install` | `yarn add` | `bun add` |
| 跑脚本 | `pnpm run` | `npm run` | `yarn` | `bun run` |
| 执行二进制 | `pnpm exec` | `npm exec` | `yarn exec` | `bunx` |

⚠️ Bun 的 dev 标志是小写 `-d`，其他都是大写 `-D`。

---

## 通用执行模板（阶段 4.1 用）

```bash
# 1. 切到父目录
cd "${PROJECT_ROOT_PARENT}"

# 2. 跑脚手架命令（取自上面对应章节）
<根据 framework 替换>

# 3. 进入项目
cd "<project-name>"

# 4. 如脚手架未自动装依赖，手动装
<pm> install
```

阶段 4 调用 `Bash` 时把每条命令**单独**执行，逐条检查 exit code，便于精准报错。
