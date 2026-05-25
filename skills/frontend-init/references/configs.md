# 工程化配置文件模板库

阶段 4.3 写文件时用这些模板。每个模板都是**当前推荐写法**（2025 年）：ESLint flat config、Prettier 3、Husky v9+、commitlint v19+。

按需替换 `<占位符>`。

---

## 1. ESLint flat config (`eslint.config.js`)

### 1.1 通用基线（任何项目都能用）

```js
import js from '@eslint/js';
import tseslint from 'typescript-eslint';

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  {
    ignores: ['dist/**', 'build/**', 'node_modules/**', '.next/**', '.nuxt/**', '.svelte-kit/**'],
  },
];
```

**对应依赖**：
```
eslint @eslint/js typescript-eslint
```

### 1.2 React 项目追加

```js
import react from 'eslint-plugin-react';
import reactHooks from 'eslint-plugin-react-hooks';

// 加入 export default 数组：
{
  files: ['**/*.{jsx,tsx}'],
  plugins: { react, 'react-hooks': reactHooks },
  rules: {
    ...react.configs.recommended.rules,
    ...reactHooks.configs.recommended.rules,
    'react/react-in-jsx-scope': 'off',
  },
  settings: { react: { version: 'detect' } },
}
```

**追加依赖**：`eslint-plugin-react eslint-plugin-react-hooks`

### 1.3 Vue 项目追加

```js
import vue from 'eslint-plugin-vue';
import vueParser from 'vue-eslint-parser';

// 加入数组：
...vue.configs['flat/recommended'],
{
  files: ['**/*.vue'],
  languageOptions: { parser: vueParser, parserOptions: { parser: tseslint.parser } },
}
```

**追加依赖**：`eslint-plugin-vue vue-eslint-parser`

### 1.4 Svelte 项目追加

```js
import svelte from 'eslint-plugin-svelte';

...svelte.configs['flat/recommended'],
```

**追加依赖**：`eslint-plugin-svelte`

### 1.5 Next.js 项目特殊处理

Next.js 内置 ESLint 时默认是老式 `.eslintrc`，要用 flat 需手动迁移。**新建模式下保留 Next 默认**，增强模式下问用户是否迁移。

---

## 2. Prettier 3 (`.prettierrc.json`)

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

如选了 Tailwind，追加 `plugins`：

```json
{
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

**对应依赖**：
```
prettier
# 选了 Tailwind 还要：prettier-plugin-tailwindcss
```

### `.prettierignore`

```
dist
build
node_modules
.next
.nuxt
.svelte-kit
coverage
pnpm-lock.yaml
package-lock.json
yarn.lock
bun.lockb
```

---

## 3. EditorConfig (`.editorconfig`)

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
indent_style = space
indent_size = 2
insert_final_newline = true
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

---

## 4. Husky v9+ 配置

### 4.1 初始化

```bash
<pm> add -D husky
<pm> exec husky init
```

`husky init` 会自动：
- 创建 `.husky/pre-commit`（默认内容 `npm test`）
- 在 `package.json` 加 `"prepare": "husky"`

### 4.2 `.husky/pre-commit`

直接覆盖：

```sh
<pm> exec lint-staged
```

### 4.3 `.husky/commit-msg`

如选了 commitlint：

```sh
<pm> exec --no -- commitlint --edit "$1"
```

> ⚠️ Husky v9+ 不再需要 `#!/usr/bin/env sh` shebang 和 `. "$(dirname -- "$0")/_/husky.sh"` 引导行，直接写命令即可。

---

## 5. lint-staged（`package.json` 字段）

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx,vue,svelte}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml,yaml,css,scss}": [
      "prettier --write"
    ]
  }
}
```

**对应依赖**：`lint-staged`

---

## 6. commitlint (`commitlint.config.js`)

```js
export default {
  extends: ['@commitlint/config-conventional'],
};
```

**对应依赖**：`@commitlint/cli @commitlint/config-conventional`

---

## 7. GitHub Actions CI (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9
        # npm/yarn 项目把这块换成对应 setup

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - run: pnpm run lint
      - run: pnpm run format:check
      - run: pnpm run build
      # 如装了 Vitest：
      # - run: pnpm exec vitest run
```

> 按用户包管理器替换 `pnpm`：
> - npm: `npm ci` + `actions/setup-node` 的 `cache: npm`
> - yarn: `yarn install --frozen-lockfile` + `cache: yarn`
> - bun: 用 `oven-sh/setup-bun@v2`

---

## 8. `.gitignore` 标准模板（如脚手架未生成）

```
# dependencies
node_modules
.pnp
.pnp.js

# build outputs
dist
build
.next
.nuxt
.svelte-kit
.angular
.output
out

# logs
*.log
npm-debug.log*
yarn-debug.log*
pnpm-debug.log*

# env
.env
.env.local
.env.*.local

# editor
.vscode/*
!.vscode/settings.json
!.vscode/extensions.json
.idea
.DS_Store

# coverage
coverage
.nyc_output

# misc
*.local
```

---

## 9. README 模板

```markdown
# <project-name>

> <一句话项目描述>

## 开发

\`\`\`bash
<pm> install
<pm> dev
\`\`\`

## 构建

\`\`\`bash
<pm> build
\`\`\`

## 脚本

| 命令 | 作用 |
|------|------|
| `<pm> dev` | 开发服务器 |
| `<pm> build` | 生产构建 |
| `<pm> lint` | 代码检查 |
| `<pm> format` | 格式化全部文件 |

## 技术栈

- <framework>
- <package_manager>
- <根据用户勾选的工程化清单列出>
```

---

## 10. TypeScript strict（如脚手架未启用）

修改 `tsconfig.json`，在 `compilerOptions` 加：

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

用 Edit 工具合并到现有 tsconfig，**不要**全文覆盖（脚手架的 tsconfig 通常带必要的 paths/include 配置）。

---

## 11. Tailwind CSS 4

Tailwind 4 安装方式因框架而异，参考[官方文档](https://tailwindcss.com/docs/installation)。常见路径：

### Vite

```bash
<pm> add tailwindcss @tailwindcss/vite
```

`vite.config.ts`：

```ts
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [tailwindcss(), /* 其他插件 */],
});
```

入口 CSS：

```css
@import "tailwindcss";
```

### Next.js / Nuxt / SvelteKit

走 PostCSS 路径，安装 `tailwindcss @tailwindcss/postcss postcss`，按各框架文档配。

阶段 4 时**优先**让脚手架自带 Tailwind（Next.js 用 `--tailwind`，create-vue 用对应交互项），实在没有再走手动安装。

---

## 12. UI 组件库（按框架分章节）

阶段 2 用户选了 UI 库后，按对应章节走：**安装 → 入口集成 → 与 Tailwind 共存检查**。

### 12.1 React 生态

#### shadcn/ui（推荐，复制式无 npm 包）

```bash
<pm> dlx shadcn@latest init
# 交互问：style (default/new-york)、base color、CSS variable mode、components.json 路径
<pm> dlx shadcn@latest add button card dialog  # 按需添加组件
```

**注意**：shadcn 不是 npm 包，是把组件源码复制到你的 `src/components/ui/` 目录，所以**必须先有 Tailwind**。阶段 2 如未勾 Tailwind，强制提示用户先勾 Tailwind。

#### Ant Design 5 / 6（**入口模板 v2，2026-05 修订**）

```bash
<pm> add antd
```

入口 `src/main.tsx` / `src/index.tsx` **完整模板**（含 ConfigProvider + AntdApp + 高度兜底）：

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { ConfigProvider, App as AntdApp } from 'antd';
import zhCN from 'antd/locale/zh_CN';
import App from '@/App';
import '@/index.css';

const container = document.getElementById('root');
if (!container) {
  throw new Error('Root container not found');
}

createRoot(container).render(
  <StrictMode>
    <ConfigProvider locale={zhCN} theme={{ token: { colorPrimary: '#1677ff' } }}>
      {/* ⚠️ AntdApp 渲染时插入 <div class="ant-app"> 包裹层，默认无 height: 100%
          会把高度链断在 #root 和业务 layout 之间。必须给 className + style 撑满，
          并配合 src/index.css 全局 .ant-app { height: 100% } 兜底（见 §12.8） */}
      <AntdApp className="h-full" style={{ height: '100%' }}>
        <App />
      </AntdApp>
    </ConfigProvider>
  </StrictMode>,
);
```

**关键说明**：

| 概念 | 用途 / 副作用 |
|------|------------|
| `ConfigProvider` | 注入 antd 主题 token + locale；包住所有用 antd 的子树 |
| `AntdApp`（`import { App } from 'antd'`） | 提供 `message.useApp() / notification.useApp() / Modal.useApp()` 上下文。**替代被弃用的静态 `message.success(...)` 调用**（静态调用脱离 ConfigProvider 上下文，主题/locale 不生效） |
| `<div class="ant-app">` | AntdApp 实际渲染的 DOM 包裹层。**默认 height: auto**，会把高度链断在这里 → 全屏布局塌陷。**必须给它 height: 100%**（className + style + 全局 CSS 三重兜底） |

**与 Tailwind 共存（v2 修订）**：antd 5/6 用 CSS-in-JS，无需 reset；但 Tailwind 的 `preflight` 会重置 `button` 默认样式。**推荐做法**：跳过 preflight（`@import 'tailwindcss/theme.css' + 'utilities.css'` 而不是 `@import "tailwindcss"`）+ 补 §12.8 的 unlayered reset（**含 `.ant-app` 兜底**）。

> ❌ **错误示范**（之前 v1 模板写的，结果布局塌陷）：
> ```tsx
> <ConfigProvider><AntdApp><App /></AntdApp></ConfigProvider>
> ```
> 没给 AntdApp 任何 height 提示 → `<div class="ant-app">` 高度 auto → 全屏布局崩。

#### Material-UI v6

```bash
<pm> add @mui/material @emotion/react @emotion/styled
# 可选图标包：
<pm> add @mui/icons-material
```

入口加 `ThemeProvider` + `CssBaseline`：

```tsx
import { ThemeProvider, createTheme, CssBaseline } from '@mui/material';
const theme = createTheme({ palette: { mode: 'light' } });

<ThemeProvider theme={theme}>
  <CssBaseline />
  <App />
</ThemeProvider>
```

#### Mantine v7

```bash
<pm> add @mantine/core @mantine/hooks @emotion/react
```

入口：

```tsx
import { MantineProvider } from '@mantine/core';
import '@mantine/core/styles.css';

<MantineProvider><App /></MantineProvider>
```

#### Chakra UI v3

```bash
<pm> add @chakra-ui/react @emotion/react
```

入口：

```tsx
import { ChakraProvider, defaultSystem } from '@chakra-ui/react';
<ChakraProvider value={defaultSystem}><App /></ChakraProvider>
```

⚠️ Chakra v3 API 与 v2 差异大，**不要**装 v2 的教程示例。

---

### 12.2 Vue 生态

#### Element Plus

```bash
<pm> add element-plus
# 推荐自动按需引入：
<pm> add -D unplugin-vue-components unplugin-auto-import
```

`main.ts`（全量）：

```ts
import ElementPlus from 'element-plus';
import 'element-plus/dist/index.css';
app.use(ElementPlus);
```

或 `vite.config.ts` 按需（推荐，bundle 小）：

```ts
import AutoImport from 'unplugin-auto-import/vite';
import Components from 'unplugin-vue-components/vite';
import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';

plugins: [
  AutoImport({ resolvers: [ElementPlusResolver()] }),
  Components({ resolvers: [ElementPlusResolver()] }),
]
```

#### Ant Design Vue 4

```bash
<pm> add ant-design-vue
```

```ts
import Antd from 'ant-design-vue';
app.use(Antd);
```

#### Naive UI

```bash
<pm> add naive-ui
# Naive UI 不需要注册，按需 import 即可。
```

```vue
<script setup>
import { NButton } from 'naive-ui';
</script>
<template><NButton>Click</NButton></template>
```

#### Vuetify 3

```bash
<pm> add vuetify
<pm> add -D vite-plugin-vuetify
```

`vite.config.ts`：

```ts
import vuetify from 'vite-plugin-vuetify';
plugins: [vue(), vuetify({ autoImport: true })]
```

`main.ts`：

```ts
import 'vuetify/styles';
import { createVuetify } from 'vuetify';
app.use(createVuetify());
```

#### PrimeVue

```bash
<pm> add primevue @primeuix/themes
```

```ts
import PrimeVue from 'primevue/config';
import Aura from '@primeuix/themes/aura';
app.use(PrimeVue, { theme: { preset: Aura } });
```

#### shadcn-vue

```bash
<pm> dlx shadcn-vue@latest init
<pm> dlx shadcn-vue@latest add button card
```

同 React 版 shadcn 注意点，**必须先装 Tailwind**。

---

### 12.3 Svelte 生态

#### shadcn-svelte

```bash
<pm> dlx shadcn-svelte@latest init
<pm> dlx shadcn-svelte@latest add button card
```

必须先装 Tailwind。

#### Bits UI（headless 原语）

```bash
<pm> add bits-ui
```

无 Provider，按需 `import` 即可。常与 Tailwind 搭配做自定义 UI。

#### Skeleton UI

```bash
<pm> add -D @skeletonlabs/skeleton @skeletonlabs/tw-plugin
```

`tailwind.config.ts`：

```ts
import { skeleton } from '@skeletonlabs/tw-plugin';
plugins: [skeleton({ themes: { preset: ['skeleton'] } })]
```

#### Flowbite Svelte

```bash
<pm> add -D flowbite-svelte flowbite
```

`tailwind.config.ts` 的 `content`：

```ts
content: ['./src/**/*.{html,js,svelte,ts}', './node_modules/flowbite-svelte/**/*.{html,js,svelte,ts}']
```

---

### 12.4 Angular 生态

#### Angular Material

```bash
<pm> dlx -p @angular/cli ng add @angular/material
# 交互问：theme、typography、animations。ng add 会自动改 app.config.ts 和 index.html。
```

#### PrimeNG

```bash
<pm> add primeng @primeuix/themes
```

`app.config.ts`：

```ts
import { providePrimeNG } from 'primeng/config';
import Aura from '@primeuix/themes/aura';

export const appConfig: ApplicationConfig = {
  providers: [providePrimeNG({ theme: { preset: Aura } })],
};
```

#### ng-zorro-antd

```bash
<pm> dlx -p @angular/cli ng add ng-zorro-antd
# 交互问：locale、theme、template，自动配 polyfills 和 app.module.ts
```

---

### 12.5 Solid 生态

#### Solid UI（shadcn-style）

```bash
<pm> dlx solidui-cli@latest init
<pm> dlx solidui-cli@latest add button
```

必须先装 Tailwind。

#### Kobalte（headless）

```bash
<pm> add @kobalte/core
```

无 Provider，按需 import。

#### Hope UI

```bash
<pm> add @hope-ui/solid @stitches/core solid-transition-group
```

```tsx
import { HopeProvider } from '@hope-ui/solid';
<HopeProvider><App /></HopeProvider>
```

---

### 12.6 通用 / Tailwind 友好

#### daisyUI（Tailwind 插件，跨框架可用）

```bash
<pm> add -D daisyui@latest
```

Tailwind 4 用 `@plugin` 语法（CSS 端）：

```css
@import "tailwindcss";
@plugin "daisyui";
```

Tailwind 3 旧写法（`tailwind.config.js`）：

```js
plugins: [require('daisyui')]
```

无 JS 入口集成，直接用 `class="btn btn-primary"`。

---

### 12.7 决策矩阵速查（阶段 2 展示菜单时用）

| 框架 | 推荐默认 | 不展示给该框架的库 |
|------|---------|-----------------|
| React | shadcn/ui（含 Tailwind）/ Ant Design 5（中后台） | Vue 库、Angular 库、Solid UI、Svelte 库 |
| Vue | Element Plus（中后台）/ Naive UI（TS 友好） | React 库、Angular 库、Solid UI、Svelte 库 |
| Svelte | shadcn-svelte（含 Tailwind） | React/Vue/Angular/Solid 库 |
| Angular | Angular Material / ng-zorro-antd | 非 Angular 库 |
| Solid | Kobalte（headless） | 非 Solid 库 |
| vanilla / 无框架 | daisyUI（如选了 Tailwind） | 所有框架专用库 |

---

### 12.8 通用注意点

1. **shadcn 系**（React/Vue/Svelte/Solid）都要先有 Tailwind，是 init 模式不是 npm 包，第一次跑 `init` 会问 4-6 个交互项
2. **antd / element-plus**：选了它们 + 又选了 Tailwind，要警告用户「Tailwind preflight 会清掉默认 button 样式」。推荐做法（Tailwind 4）：在入口 CSS **只导入 theme + utilities，跳过 preflight**，**并补最小 reset**。

   ⚠️ **2026-05 修订（v2 修复关键 bug）**：v1 把 reset 包在 `@layer base { ... }` 里，**这是错的反模式**——因为没导入 `tailwindcss/base.css`，所以 `base` 这个 cascade layer 从未在文档中被声明 → 浏览器视其为「未声明 layer」，优先级**低于**一切 unlayered 样式 + 已声明 layer。结果 reset 失效，`html { height: 100% }` 拿不到，全屏布局塌成内容自适应高度。**v2 必须把 reset 放在 `@layer` 外，作为 unlayered 拿到最高优先级**：

   ```css
   @import 'tailwindcss/theme.css' layer(theme);
   @import 'tailwindcss/utilities.css' layer(utilities);

   /* ⚠️ 最小 reset 必须 unlayered（不能放进 @layer base）
      理由：未声明的 cascade layer 优先级最低；放 @layer 外 = unlayered = 最高优先级。 */

   *, *::before, *::after {
     box-sizing: border-box;
   }

   html, body, #root {
     height: 100%;
     margin: 0;
     padding: 0;
   }

   body {
     font-family:
       -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
       'Helvetica Neue', sans-serif;
     -webkit-font-smoothing: antialiased;
     -moz-osx-font-smoothing: grayscale;
   }

   /* ⚠️ 关键兜底：AntdApp 组件（import { App } from 'antd'）会在 DOM 中
      插入一个 <div class="ant-app"> 包裹层，默认无 height: 100%，
      会把 #root → AdminLayout 的高度链断在中间，子级 layout 撑不开。
      强制撑满，配合入口 <AntdApp className="h-full" style={{ height: '100%' }}> 双保险。 */
   .ant-app {
     height: 100%;
   }
   ```

   而不是 `@import "tailwindcss"`（这会带上 preflight）。
   **强制提示**：阶段 4.3 写 `src/index.css` 时，凡是"选了 Tailwind + 选了 antd/element-plus"的组合，**必须**包含上面整段 unlayered reset（**含 `.ant-app` 兜底**），否则全屏后台/Dashboard 类布局会出现四周白边 + 高度塌陷。

   **🔍 验证方法（DevTools，必跑）**：起 dev 后打开浏览器 DevTools → Elements 面板逐层检查高度链：

   | 节点 | computed height 应是 | 如果是 `auto` 说明 |
   |------|-------------------|------------------|
   | `<html>` | 视口高度（如 1024px） | reset 没生效（多半被错放进 layer 里） |
   | `<body>` | 同上 | 同上 |
   | `<div id="root">` | 同上 | 同上 |
   | **`<div class="ant-app">`** | **同上** | **`.ant-app` 兜底没写，或 AntdApp 没传 style** |
   | `<section class="ant-layout">` | 同上 | AdminLayout 用了 `min-h-screen` 而非 `h-full` + inline style |

   任何一层是 `auto`，下面全部塌陷。**特别注意 `<div class="ant-app">`——这是最容易被遗忘的一层**，上一轮就栽在这里。
3. **MUI / Vuetify**：Provider 必须包裹 App 根，否则主题不生效
4. **CSS-in-JS 库**（MUI / Mantine / Chakra）与 Tailwind 共存通常无冲突，但 className 优先级要注意 `important` 配置
5. **Tailwind 4 + UI 库插件**：daisyUI 等通过 `@plugin` 语法集成，**不再**用 tailwind.config.js（Tailwind 4 默认无 config 文件）

---

## 13. 项目布局模板库（按"框架 × UI库 × 布局"三元组查）

阶段 2 用户选了「布局类型」后，按本节查对应小节，拿 `src/App.tsx`（或 `App.vue` / `+layout.svelte`）+ 必要路由依赖 + 入口集成代码。

**降级策略**：如果用户选的 `(框架, UI库)` 组合在本节**没有**对应模板，自动降级展示「空白起点」或要求用户选"自定义描述"，不要硬塞别的库的模板（会编译失败）。

**与 UI 库的强耦合**：本节**只覆盖最主流的几种 UI 库 + 框架组合**（React + Antd / React + shadcn / React + 无库 / Vue + Element Plus / Vue + 无库），其他组合**渐进补**或走自定义路径。

---

### 13.0 空白起点（所有框架默认）

**React**（`src/App.tsx`）：

```tsx
export default function App() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <h1 className="text-3xl font-bold">Hello, world.</h1>
    </div>
  );
}
```

**Vue**（`src/App.vue`）：

```vue
<template>
  <div class="flex min-h-screen items-center justify-center">
    <h1 class="text-3xl font-bold">Hello, world.</h1>
  </div>
</template>
```

无需路由库。

---

### 13.1 后台管理（Admin Dashboard）

布局：Header（顶部固定）+ Sidebar（左侧菜单 + 折叠按钮）+ Content（路由出口）+ Footer（可选）。

#### 13.1.1 React + Antd（**多文件目录结构 v2，2026-05 修订**）

> **🔥 v2 修订原因**：v1 版本把 layout/pages/router 全堆在单文件 App.tsx，不符合后台项目工程实践。
> 实跑发现后台项目应**至少**分 `layouts/` + `pages/` + `router/` + `components/` 四个目录，否则页面一多就崩。
> **同时要求** `src/index.css` 含跳 preflight + 最小 reset 代码块（见 §12.8），否则布局不撑满。

**追加依赖**：

```bash
<pm> add @ant-design/icons react-router-dom
```

**目标目录结构**：

```
src/
├── App.tsx                  # 仅 <RouterProvider router={router} />
├── index.tsx                # ConfigProvider + AntdApp 包裹（见 §12.2）
├── index.css                # Tailwind 分层 + 最小 reset（见 §12.8）
├── layouts/
│   └── AdminLayout.tsx      # Sider + Header + Content + <Outlet />
├── pages/
│   ├── Dashboard.tsx
│   ├── Users.tsx
│   └── Settings.tsx
├── router/
│   └── index.tsx            # createBrowserRouter([{ element: <AdminLayout/>, children: [...] }])
└── components/              # 预留（.gitkeep 占位）
```

**`src/App.tsx`**：

```tsx
import { RouterProvider } from 'react-router-dom';
import { router } from '@/router';

export default function App() {
  return <RouterProvider router={router} />;
}
```

**`src/router/index.tsx`**（React Router v7 数据路由）：

```tsx
import { createBrowserRouter, Navigate } from 'react-router-dom';
import AdminLayout from '@/layouts/AdminLayout';
import Dashboard from '@/pages/Dashboard';
import Users from '@/pages/Users';
import Settings from '@/pages/Settings';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <AdminLayout />,
    children: [
      { index: true, element: <Navigate to="/dashboard" replace /> },
      { path: 'dashboard', element: <Dashboard /> },
      { path: 'users', element: <Users /> },
      { path: 'settings', element: <Settings /> },
    ],
  },
]);
```

**`src/layouts/AdminLayout.tsx`**（**注意：用 `h-full` 而非 `min-h-screen`，配合 §12.8 的 `html/body/#root { height: 100% }` 撑满**）：

```tsx
import { useState } from 'react';
import { Layout, Menu, Button, theme as antdTheme } from 'antd';
import {
  MenuFoldOutlined, MenuUnfoldOutlined,
  DashboardOutlined, UserOutlined, SettingOutlined,
} from '@ant-design/icons';
import { Link, Outlet, useLocation } from 'react-router-dom';

const { Header, Sider, Content } = Layout;

const menuItems = [
  { key: '/dashboard', icon: <DashboardOutlined />, label: <Link to="/dashboard">Dashboard</Link> },
  { key: '/users', icon: <UserOutlined />, label: <Link to="/users">Users</Link> },
  { key: '/settings', icon: <SettingOutlined />, label: <Link to="/settings">Settings</Link> },
];

export default function AdminLayout() {
  const [collapsed, setCollapsed] = useState(false);
  const location = useLocation();
  const { token: { colorBgContainer, borderRadiusLG } } = antdTheme.useToken();

  return (
    // ⚠️ 双保险：className + inline style 同时给 height: 100%
    // 原因：antd Layout 自带 min-height: 0，仅靠 className 可能被 antd 自身样式覆盖
    <Layout style={{ height: '100%' }} className="h-full">
      <Sider trigger={null} collapsible collapsed={collapsed}>
        <div className="m-4 h-8 rounded bg-white/20 text-center leading-8 font-bold text-white">
          {collapsed ? 'A' : 'My Admin'}
        </div>
        <Menu theme="dark" mode="inline" selectedKeys={[location.pathname]} items={menuItems} />
      </Sider>
      <Layout>
        <Header className="flex items-center !px-4" style={{ background: colorBgContainer }}>
          <Button
            type="text"
            icon={collapsed ? <MenuUnfoldOutlined /> : <MenuFoldOutlined />}
            onClick={() => setCollapsed(!collapsed)}
          />
        </Header>
        <Content
          className="m-4 overflow-auto p-6"
          style={{ background: colorBgContainer, borderRadius: borderRadiusLG }}
        >
          <Outlet />
        </Content>
        {/* ⚠️ 是否加 Footer 由用户阶段 2 子问询决定，默认不加 */}
      </Layout>
    </Layout>
  );
}
```

**`src/pages/Dashboard.tsx`**（其余 Users / Settings 同模板，只改 title 和 paragraph）：

```tsx
import { Typography } from 'antd';

const { Title, Paragraph } = Typography;

export default function Dashboard() {
  return (
    <div>
      <Title level={2}>Dashboard</Title>
      <Paragraph type="secondary">欢迎进入后台管理系统。</Paragraph>
    </div>
  );
}
```

**关键约束**：

| 约束 | 原因 |
|------|------|
| `src/index.css` **必须**包含 §12.8 的 **unlayered** 最小 reset（**不能**放进 `@layer base`，且**必须含 `.ant-app { height: 100% }` 兜底**） | 未声明 layer 优先级最低 → reset 失效；缺 `.ant-app` 兜底 → 高度链断在 AntdApp 包裹层 |
| `src/index.tsx` 的 **AntdApp 必须传** `className="h-full"` + `style={{ height: '100%' }}` | 详见 §12.2 v2 模板。AntdApp 默认渲染一个无高度 div，不传 → 高度链断 |
| `AdminLayout` **同时**用 `className="h-full"` + `style={{ height: '100%' }}` | antd Layout 自带 `min-height: 0`，仅靠 className 可能被覆盖；inline style 优先级最高，保底 |
| **高度链完整性**：`<html>` / `<body>` / `<#root>` / `<.ant-app>` / `<.ant-layout>` **逐层** height: 100% | 任何一层 auto，下面全部塌陷。**dev 启动后必须用 DevTools 逐层 check computed height（见 §12.8 验证方法）** |
| 路由用 `createBrowserRouter` + `<RouterProvider>` | React Router v7 推荐写法，支持后续 loader/action 数据加载 |
| 业务页面**强制**分到 `src/pages/`，**不**写在 layout 文件里 | 页面一多直接 import 路径混乱 |
| 默认**不加 Footer** | 阶段 2 用户没明确要 Footer 就不加；要的话子问询单独问 |
| 顶层 `src/App.tsx` **保持极简** | 只渲染 RouterProvider，不要把 Provider 全堆在这里（Provider 在 src/index.tsx 处理） |

#### 13.1.2 React + shadcn/ui

依赖：`<pm> dlx shadcn@latest add sidebar button` + `<pm> add react-router-dom`。

shadcn 的 `Sidebar` 组件自带 `SidebarProvider` + `SidebarTrigger`，按官方文档拼装。详见 [shadcn Sidebar block](https://ui.shadcn.com/blocks#sidebar)。

#### 13.1.3 Vue + Element Plus

**装路由**：

```bash
<pm> add vue-router
```

**`src/App.vue`** 用 `<el-container>` + `<el-aside>` + `<el-header>` + `<el-main>` + `<el-menu>`，参考 [Element Plus Container 示例](https://element-plus.org/zh-CN/component/container.html)。

#### 13.1.4 其他组合

降级处理：告诉用户「该 UI 库的后台管理布局模板暂未沉淀，建议选『自定义描述』或我用『空白起点』+ 你自己拼装」。

---

### 13.2 官网 / Marketing

布局：顶部导航 + Hero（大标题 + CTA 按钮）+ Features Sections（三栏卡片）+ Footer。

#### 13.2.1 React + Antd

```tsx
import { Layout, Button, Typography, Row, Col, Card, Space } from 'antd';

const { Header, Content, Footer } = Layout;
const { Title, Paragraph } = Typography;

export default function App() {
  return (
    <Layout className="min-h-screen">
      <Header className="!bg-white shadow flex items-center justify-between">
        <div className="text-xl font-bold">My Brand</div>
        <Space>
          <Button type="text">Features</Button>
          <Button type="text">Pricing</Button>
          <Button type="primary">Get Started</Button>
        </Space>
      </Header>
      <Content>
        <section className="py-24 text-center bg-gradient-to-b from-blue-50 to-white">
          <Title>Build faster with My Brand</Title>
          <Paragraph type="secondary" className="!mb-8">
            The opinionated stack that gets you to production this week.
          </Paragraph>
          <Space>
            <Button type="primary" size="large">Start free</Button>
            <Button size="large">Read docs</Button>
          </Space>
        </section>
        <section className="py-16 px-8 max-w-6xl mx-auto">
          <Row gutter={[24, 24]}>
            {['Fast', 'Reliable', 'Open'].map((title) => (
              <Col xs={24} md={8} key={title}>
                <Card title={title}>One-line description.</Card>
              </Col>
            ))}
          </Row>
        </section>
      </Content>
      <Footer className="text-center">© 2025 My Brand</Footer>
    </Layout>
  );
}
```

#### 13.2.2 React + shadcn / Tailwind only

不依赖 UI 库版本：用纯 Tailwind + 原生标签（`<header>`/`<section>`/`<footer>`）即可。

---

### 13.3 Dashboard 数据看板

布局：Header（仅 logo + 用户头像）+ KPI 卡片 Grid（4 列）+ 图表占位区（2 列）。

#### 13.3.1 React + Antd

```tsx
import { Layout, Row, Col, Card, Statistic, Typography } from 'antd';
import { ArrowUpOutlined, UserOutlined, DollarOutlined, ShoppingOutlined } from '@ant-design/icons';

const { Header, Content } = Layout;
const { Title } = Typography;

export default function App() {
  return (
    <Layout className="min-h-screen bg-slate-50">
      <Header className="!bg-white shadow flex items-center">
        <Title level={4} className="!m-0">My Dashboard</Title>
      </Header>
      <Content className="p-6 space-y-6">
        <Row gutter={[16, 16]}>
          {[
            { title: 'Active Users', value: 1128, icon: <UserOutlined /> },
            { title: 'Revenue', value: 9381, prefix: '$', icon: <DollarOutlined /> },
            { title: 'Orders', value: 248, icon: <ShoppingOutlined /> },
            { title: 'Growth', value: 11.28, suffix: '%', icon: <ArrowUpOutlined /> },
          ].map((stat) => (
            <Col xs={24} sm={12} lg={6} key={stat.title}>
              <Card>
                <Statistic
                  title={stat.title}
                  value={stat.value}
                  prefix={stat.prefix ?? stat.icon}
                  suffix={stat.suffix}
                />
              </Card>
            </Col>
          ))}
        </Row>
        <Row gutter={[16, 16]}>
          <Col xs={24} lg={16}>
            <Card title="Trend (last 30 days)" className="h-80">
              <div className="text-slate-400 text-center pt-20">[Chart placeholder]</div>
            </Card>
          </Col>
          <Col xs={24} lg={8}>
            <Card title="Top sources" className="h-80">
              <div className="text-slate-400 text-center pt-20">[List placeholder]</div>
            </Card>
          </Col>
        </Row>
      </Content>
    </Layout>
  );
}
```

> 真要做图表，**单独提示**用户："要不要装 `recharts` / `@ant-design/charts` / `echarts-for-react`？本 skill 不内置，避免无脑塞依赖。"

---

### 13.4 文档站 (Docs)

布局：左侧 Sidebar 目录树 + 右侧 Markdown 内容 + 顶部搜索框。

> **强烈推荐**用 [VitePress](https://vitepress.dev/) / [Nextra](https://nextra.site/) / [Astro Starlight](https://starlight.astro.build/) 等**专用文档框架**而不是自己拼。本布局模板只在用户明确不想用文档框架时启用，且**默认提示一句**："文档站建议直接用 VitePress，省 80% 工作量。你坚持自拼也行，但 SSR / 搜索 / 全文检索都要自己做。"

#### 13.4.1 React + Antd（自拼版）

简化版：`Layout.Sider` + `Tree` 组件做目录，右侧 `Typography` 渲染 markdown（需额外装 `react-markdown`），顶部 `Input.Search`。完整模板略，按用户实际需求再生成。

---

### 13.5 自定义描述（用户描述布局）

阶段 2 用户选了 `0f. 自定义描述` 后，**追问一次**：

> "描述一下布局，越具体越好。例：
> - '顶部 logo+ 登录按钮，下面分两栏，左 30% 是表单，右 70% 是预览'
> - '全屏地图，右上角浮窗状态卡，底部抽屉'
> - '三栏，左侧聊天列表，中间消息流，右侧用户信息卡'
>
> 我会按你描述生成 `App.tsx`（或 Vue/Svelte 等价版本），优先用你选的 UI 库（Antd 用 Layout/Card；shadcn 用 Card/Sidebar；纯 Tailwind 用 div+flex）。"

收到后用 Read+Edit 现场写 `App.tsx`，不需要 reference 模板支撑。

---

### 13.6 布局模板覆盖矩阵（实现进度，按需补全）

| 布局 \ 组合 | React+Antd | React+shadcn | React+无库 | Vue+ElementPlus | Vue+无库 | Svelte | Angular | Solid |
|------------|-----------|-------------|-----------|-----------------|---------|--------|---------|-------|
| 空白起点 | ✅ §13.0 | ✅ §13.0 | ✅ §13.0 | ✅ §13.0 | ✅ §13.0 | ✅ §13.0 | TODO | TODO |
| 后台管理 | ✅ §13.1.1 | ⚠️ §13.1.2 | TODO | ⚠️ §13.1.3 | TODO | TODO | TODO | TODO |
| 官网 | ✅ §13.2.1 | ✅ §13.2.2 | ✅ §13.2.2 | TODO | TODO | TODO | TODO | TODO |
| Dashboard | ✅ §13.3.1 | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| 文档站 | ⚠️ §13.4.1 | TODO | TODO | TODO | TODO | TODO | TODO | TODO |
| 自定义 | ✅ §13.5 现场生成 | ✅ §13.5 | ✅ §13.5 | ✅ §13.5 | ✅ §13.5 | ✅ §13.5 | ✅ §13.5 | ✅ §13.5 |

图例：✅ 完整模板 / ⚠️ 简化提示 / TODO 暂未沉淀

**降级处理**：TODO 行用户选了 → 自动降级到「空白起点」+ 提示用户走「自定义描述」补描述。

---

## 14. Webpack 5 手工配置模板（用户坚持 React+Webpack 路径时用）

> **前置警告**（必须给用户说）：React 团队 2025-02 已 [弃用 CRA](https://react.dev/blog/2025/02/14/sunsetting-create-react-app)，推荐用 Vite/Next.js/Remix。本模板是用户**明知风险后坚持手工配 Webpack 5 时**的兜底，不是默认路径。后续 webpack/loader 升级兼容性由用户自己维护。

### 14.1 依赖

**生产**（按用户选的 UI 库追加）：

```bash
npm i react react-dom
# 选了 Antd：
npm i antd
```

**开发**（一次性装齐，**注意 eslint 锁 ^9**）：

```bash
npm i -D \
  webpack webpack-cli webpack-dev-server webpack-merge \
  typescript@^5 @types/react @types/react-dom @types/node \
  @swc/core swc-loader \
  html-webpack-plugin \
  css-loader style-loader postcss postcss-loader @tailwindcss/postcss tailwindcss \
  eslint@^9 @eslint/js@^9 typescript-eslint eslint-plugin-react eslint-plugin-react-hooks globals \
  prettier prettier-plugin-tailwindcss \
  code-inspector-plugin \
  ts-node
# 选了 Husky+commitlint：
npm i -D husky @commitlint/cli @commitlint/config-conventional
```

⚠️ **eslint 必须锁 ^9**：v10 与 `eslint-plugin-react` peer 冲突；**不**用 `--legacy-peer-deps` 绕过。
⚠️ **TypeScript 锁 ^5**：v6 与 `ts-node` 兼容性不稳定。

### 14.2 `webpack.config.ts`

**关键约束**：
- 用 `import.meta.url` polyfill 算 `__dirname`（webpack-cli 7 把 TS config 当 ESM 加载）
- **`devServer.port` 必须用 `'auto'`**，避免 3000 端口被占用时启动失败
- SWC 替代 babel-loader 提速 20×

```ts
import path from 'path';
import { fileURLToPath } from 'node:url';
import HtmlWebpackPlugin from 'html-webpack-plugin';
import { codeInspectorPlugin } from 'code-inspector-plugin';
import type { Configuration } from 'webpack';
import 'webpack-dev-server';

// ESM polyfill: webpack-cli 7 loads .ts config as ESM (no __dirname)
const __dirname = path.dirname(fileURLToPath(import.meta.url));

interface Argv {
  mode: 'development' | 'production';
}

const config = (_env: unknown, argv: Argv): Configuration => {
  const isDev = argv.mode === 'development';

  return {
    mode: argv.mode,
    entry: './src/index.tsx',
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: isDev ? '[name].js' : '[name].[contenthash:8].js',
      clean: true,
      publicPath: '/',
    },
    resolve: {
      extensions: ['.tsx', '.ts', '.jsx', '.js'],
      alias: { '@': path.resolve(__dirname, 'src') },
    },
    module: {
      rules: [
        {
          test: /\.(ts|tsx)$/,
          exclude: /node_modules/,
          use: {
            loader: 'swc-loader',
            options: {
              jsc: {
                parser: { syntax: 'typescript', tsx: true },
                transform: { react: { runtime: 'automatic', development: isDev } },
                target: 'es2020',
              },
            },
          },
        },
        { test: /\.css$/, use: ['style-loader', 'css-loader', 'postcss-loader'] },
      ],
    },
    plugins: [
      new HtmlWebpackPlugin({ template: './public/index.html' }),
      codeInspectorPlugin({ bundler: 'webpack' }),
    ],
    devServer: {
      port: process.env.PORT ? Number(process.env.PORT) : 'auto',
      hot: true,
      open: true,
      historyApiFallback: true,
    },
    devtool: isDev ? 'eval-source-map' : 'source-map',
  };
};

export default config;
```

### 14.3 `tsconfig.json`

同 §10 的 strict 模板 + 加 `ts-node` 子配置（让 ts-node 用 CJS 加载本模板的 webpack.config.ts）：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "jsx": "react-jsx",
    "isolatedModules": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },
    "types": ["node"]
  },
  "include": ["src/**/*", "webpack.config.ts"],
  "exclude": ["node_modules", "dist"],
  "ts-node": {
    "compilerOptions": { "module": "CommonJS", "moduleResolution": "Node" }
  }
}
```

### 14.4 `package.json scripts`

```json
{
  "scripts": {
    "dev": "webpack serve --mode development",
    "build": "webpack --mode production",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

如选了 Husky，追加 `"prepare": "husky"`。

### 14.5 端口策略说明（必须告知用户）

模板默认 `devServer.port: 'auto'`，含义：

- webpack-dev-server 自动找一个**可用端口**启动（通常从 8080 起递增），避免 3000 被占用时报错
- 想固定端口：跑 `cross-env PORT=4000 npm run dev`（需先 `npm i -D cross-env`）或直接改配置文件
- 启动后**实际端口看终端输出**（`Project is running at http://localhost:XXXX/`）

⚠️ **不要把 port 写死为 3000**（最常见与 Next.js / Create React App 默认值冲突），这是反模式。
