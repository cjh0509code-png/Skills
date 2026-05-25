# Generator 模板 — 模块型（Module Generator）

> **适用范围**：生成纯逻辑模块，无 UI 视图。
> 输出形态：`.ts` 文件（含函数 / 类 / 常量 / Hook / Composable / Slice）+ `.test.ts`
> 典型场景：React 自定义 Hook（`useXxx`）、Vue Composable（`useXxx`）、Redux/Zustand Slice、util 工具函数、API client、constants 模块
>
> **不适用**：生成 UI 组件 → 用 `template-generator-component.md`
>
> 用访谈中收集的信息替换所有 `[占位符]`，输出完整 SKILL.md 给用户。

---

## 决策树

填充模板前先确认（这些都已在 Q3-module 收集）：

1. **模块种类**：React Hook / Vue Composable / Redux Slice / Zustand Slice / Util / API client / Constants
2. **命名前缀**：`useXxx`（Hook/Composable）/ `xxxSlice`（Slice）/ 无（util/constants）
3. **输出位置**：`src/hooks/` / `src/composables/` / `src/store/slices/` / `src/utils/` / `src/api/`
4. **文件数**：单 `.ts` 还是文件夹（含 `.test.ts` + `index.ts`）
5. **响应式依赖**：Vue Composable 需 `ref/computed/lifecycle`；React Hook 需 `useState/useEffect/useCallback`；util 不需要
6. **副作用清理**：Hook/Composable 必须正确清理（`onUnmounted` / `useEffect` cleanup）

---

## 完整模板

```markdown
---
name: [技能名，小写连字符，如 gen-hook / gen-composable / gen-slice]
description: |
  [一句话说清楚生成什么模块、什么场景用]。
  [补充 2-3 个推销式触发场景]。
  触发：/[技能名] <[参数]>，或"[触发词1]"、"[触发词2]"
triggers:
  - [技能名]
allowed-tools: [按 target_agents 决定，见 agent-compatibility.md §2：Universal 用 Read,Write,Glob,Bash；Claude Code Only 加 Task]
---

# [技能名] — [中文描述]

## 我做什么

[输入 → 输出 → 解决什么痛点]

例：
> 输入 `/gen-composable useDebounce` → 在 `src/composables/useDebounce/` 下生成
> `useDebounce.ts`（含 `ref` + `watch` + `onUnmounted` 清理）、`useDebounce.test.ts`、`index.ts`。
> 解决：重复手写防抖逻辑、忘记清理 timer 导致内存泄漏。

## 命令格式

| 用法 | 说明 |
|------|------|
| `/[技能名] <name>` | [生成什么] |
| `/[技能名] <name> --[选项]` | [可选行为] |

## 技术栈适配

- 框架：[React / Vue / Svelte / 无（util）]
- 语言：[TypeScript 严格模式 / JavaScript]
- 测试：[Vitest / Jest]
- 框架专有约束（来自 Q4 follow-up）：
  - [Vue：`<script setup>` / Composition API / Pinia 等]
  - [React：React 18+ / Concurrent Features 等]

## 执行流程

> **跨平台原则**：本技能不依赖 bash heredoc 或 POSIX 特有命令。所有逻辑由 Claude 用原生工具（Glob / Read / Write / Edit / Grep）执行；只在调用 `npx tsc` / `npx vitest` 这类已经跨平台的 CLI 时才用 Bash 工具。

### 步骤 1：解析参数 + 校验命名

**Claude 执行**（无需调用 shell）：
1. 从用户输入提取 `<name>` 参数
2. 校验命名匹配规则（按 Q3-module 收集到的规则）：
   - 例：Hook/Composable 必须匹配 `^use[A-Z][a-zA-Z0-9]*$`
   - 例：Slice 必须匹配 `^[a-z][a-zA-Z0-9]*Slice$`
3. 若 `<name>` 为空或不匹配规则，立刻回应清晰错误并停止：
   > "❌ 命名 `xxx` 不符合规则 `^use[A-Z]...$`，例如：`useDebounce`、`useFetch`"

### 步骤 2：检查目标位置

**Claude 执行**：
1. 用 **Glob 工具**搜索 `[输出目录]/<name>/**/*`
2. 若有结果（目标已存在）：按 Q3 决策处理
   - 报错：告知路径已存在，请改名重试
   - 加序号：自动用 `<name>2`、`<name>3`
   - 询问：列出现有文件 + 询问覆盖

### 步骤 3：生成文件

[列出生成哪些文件，例如：]
- `src/composables/<name>/<name>.ts`
- `src/composables/<name>/<name>.test.ts`
- `src/composables/<name>/index.ts`

**`<name>.ts` 模板**（按 Q1 框架选）：

#### Vue Composable 示范
```ts
import { ref, computed, onMounted, onUnmounted, type Ref } from 'vue'

/**
 * [描述这个 Composable 做什么]
 * @example
 *   const { state, action } = use[Name]()
 */
export function use[Name](/* params */) {
  // 1. 响应式状态
  const state: Ref<unknown> = ref(null)

  // 2. 派生值
  const derived = computed(() => state.value)

  // 3. 副作用（按需）
  let timer: ReturnType<typeof setTimeout> | null = null

  // 4. 生命周期
  onMounted(() => {
    // setup
  })

  onUnmounted(() => {
    // 必须清理：timer / event listener / subscription
    if (timer) clearTimeout(timer)
  })

  // 5. 返回 API
  return {
    state: state as Readonly<typeof state>,  // 推荐 readonly
    derived,
    // action(): ...
  }
}
```

#### React Hook 示范
```ts
import { useState, useEffect, useCallback, useRef } from 'react'

/**
 * [描述这个 Hook 做什么]
 * @example
 *   const { state, action } = use[Name]()
 */
export function use[Name](/* params */) {
  // 1. 状态
  const [state, setState] = useState<unknown>(null)

  // 2. ref（保存可变值）
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  // 3. 回调（memoized）
  const action = useCallback(() => {
    // ...
  }, [/* deps */])

  // 4. 副作用 + 清理
  useEffect(() => {
    return () => {
      if (timerRef.current) clearTimeout(timerRef.current)
    }
  }, [])

  return { state, action }
}
```

#### Redux Toolkit Slice 示范
```ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit'

interface [Name]State {
  // ...
}

const initialState: [Name]State = {
  // ...
}

const [name]Slice = createSlice({
  name: '[name]',
  initialState,
  reducers: {
    setSomething(state, action: PayloadAction<unknown>) {
      // ...
    },
  },
})

export const { setSomething } = [name]Slice.actions
export default [name]Slice.reducer  // ⚠️ slice 是 default export 的例外
```

#### Util 示范
```ts
/**
 * [函数用途]
 * @param input - [描述]
 * @returns [描述]
 * @example
 *   [name](something) // => result
 */
export function [name]<T>(input: T): T {
  // 实现
  return input
}
```

**`<name>.test.ts` 模板**（按 Q1 测试库选）：

```ts
import { describe, it, expect, vi } from 'vitest'
import { [name] } from './[name]'

describe('[name]', () => {
  it('initial state should be ...', () => {
    const result = [name]()
    expect(result.state).toBe(/* expected */)
  })

  it('cleanup runs on unmount (for Hook/Composable)', () => {
    // 验证清理逻辑
  })

  // 至少 1 个 happy path + 1 个 cleanup/edge case
})
```

**`index.ts` 模板**：

```ts
export { [name] } from './[name]'
// 如有类型对外暴露：
export type { [Name]Options } from './[name]'
```

### 步骤 4：输出确认

```
✅ 已生成 <name>
  ├── [输出目录]/<name>/<name>.ts
  ├── [输出目录]/<name>/<name>.test.ts
  └── [输出目录]/<name>/index.ts

下一步：
  • 运行 `pnpm test <name>` 验证测试通过
  • 在使用处导入：import { <name> } from '@/[输出目录]/<name>'
```

---

## 严格遵守的约定

[直接来自 Q4 + framework-specific 答案]

- **命名**：[按 Q4 收集的规则，如 Hook/Composable 必须 useXxx]
- **导出方式**：named export 优先（Slice 例外，需要 default 导出 reducer）
- **类型严格**：
  - 禁 `any`，必须 `unknown` 或具体类型
  - 返回值类型显式标注（不靠 inference 推断 public API）
- **依赖**：不引入新依赖，只用框架内置 API（如 Vue 用 `vue` 包、React 用 `react` 包）
- **响应式约束**（Hook/Composable 特有）：
  - Vue Composable 必须在 setup 阶段或另一个 Composable 内调用
  - React Hook 必须遵守 Rules of Hooks（顶层、非条件、非循环）
- **副作用清理**（Hook/Composable 特有）：
  - 所有 timer / event listener / subscription 必须在 cleanup 中清理
  - Vue：`onUnmounted` / `onBeforeUnmount`
  - React：`useEffect` 的 return 函数

---

## 满分作业参考

[来自 Q4 的参考样例 + 提取的风格特征]

```ts
[参考样例完整代码，或符合该项目风格的示范]
```

---

## 遇到问题时

| 情况 | 处理方式 |
|------|---------|
| 参数缺失/命名不符 | 报错 + 给出正确用法示例 |
| 目标路径已存在 | 按 Q3 决策处理 |
| 父目录不存在 | 自动 `mkdir -p`，告知用户 |
| 不在项目根（无 `package.json`）| 报错：请 cd 到项目根再试 |
| 框架依赖缺失（如 Vue 项目但 `node_modules` 无 `vue`）| 报错并提示 `pnpm install` |
| 被要求生成范围外的东西（如 UI 组件）| 礼貌说明边界，建议用 `gen-component` |

---

## 自测清单

- [ ] **标准生成**：典型参数 → 生成全部文件 → `tsc --noEmit` 通过 → `eslint` 0 error
- [ ] **测试可跑**：生成的 `.test.ts` 直接 `vitest run` 应该通过
- [ ] **命名校验**：错误命名应被拒绝并给出正确格式提示
- [ ] **副作用清理**（Hook/Composable）：生成的代码必须含 cleanup 段，否则 ESLint `react-hooks/exhaustive-deps` 或 Vue lint 会挂
- [ ] **类型完整**：`tsc --noEmit` 在 strict 模式下应零错误
```
