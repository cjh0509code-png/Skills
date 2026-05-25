# Generator 模板 — 组件型（Component Generator）

> **适用范围**：生成 UI 组件，含视图 / 样式 / Story / 测试。
> 输出形态：`.tsx` / `.vue` / `.svelte` + `.test.*` + 可选 `.stories.*` + `index.ts`
> 典型场景：Button、Modal、FormField、Card、Table 等可视化组件
>
> **不适用**：生成 Hook / Composable / Slice / util / API client → 用 `template-generator-module.md`
>
> 用访谈中收集的信息替换所有 `[占位符]`，输出完整 SKILL.md 给用户。

---

## 决策树

填充模板前先确认：

1. **触发命令格式**：`/[技能名] <参数1> [参数2]` 还是 `/[技能名] --name X --type Y`
2. **输出位置**：固定目录（如 `src/components/`）还是用户指定
3. **是否多文件**：单文件还是文件夹（含 `index` + `.test` + `.stories` + `types`）
4. **是否覆盖**：目标已存在时报错、覆盖还是追加序号

---

## 完整模板

```markdown
---
name: [技能名，小写连字符，如 gen-component]
description: |
  [一句话说清楚生成什么、什么场景用]。
  [补充 2-3 个推销式触发场景，明确告诉 Claude 何时主动用]。
  触发：/[技能名] <[参数]>，或"[触发词1]"、"[触发词2]"
triggers:
  - [技能名]
allowed-tools: [按 target_agents 决定，见 agent-compatibility.md §2：Universal 用 Read,Write,Glob,Bash；Claude Code Only 加 Task]
---

# [技能名] — [中文描述]

## 我做什么

[用一段话说清楚：输入什么 → 输出什么文件 → 解决什么痛点]

## 命令格式

| 用法 | 说明 |
|------|------|
| `/[技能名] [必填参数]` | [生成什么] |
| `/[技能名] [必填参数] --[选项]` | [可选行为] |

## 技术栈适配

- 框架：[React / Vue / Svelte / ...]
- 语言：[TypeScript / JavaScript]
- 样式：[Tailwind / CSS Modules / styled-components / ...]
- 测试：[Vitest / Jest / ...]

## 执行流程

> **跨平台原则**：本技能不依赖 bash heredoc 或 POSIX 特有命令。所有逻辑由 Claude 用原生工具（Glob / Read / Write / Edit / Grep）执行；只在调用 `npx tsc` / `npx eslint` 这类已经跨平台的 CLI 时才用 Bash 工具。

### 步骤 1：解析参数 + 校验

**Claude 执行**（无需调用 shell）：
1. 从用户输入提取参数：第一个 positional 参数视为 `<Name>`，`--key value` 形式视为选项
2. 校验：
   - 必填参数齐全（如 `<Name>` 不能为空）
   - 命名匹配 `[PascalCase/camelCase 规则]`（用 JS regex 在脑中校验）
3. 若违反，立刻用清晰错误消息回应并停止：
   > "❌ 命名 `xxx` 不符合 `[规则]`，例如：`Button`、`UserCard`"

### 步骤 2：检查目标位置

**Claude 执行**：
1. 用 **Glob 工具**搜索目标路径，例如 `src/components/<Name>/**/*`
2. 若有结果（说明目标已存在）：按 Q3 的策略处理
   - 报错策略：告知用户路径已存在，请改名后重试
   - 加序号策略：自动用 `<Name>2`、`<Name>3` ……
   - 询问策略：列出现有文件 + 询问是否覆盖

### 步骤 3：生成文件树

[列出会生成哪些文件，例如：]
- `src/components/<Name>/<Name>.tsx`
- `src/components/<Name>/<Name>.test.tsx`
- `src/components/<Name>/<Name>.stories.tsx`
- `src/components/<Name>/index.ts`

每个文件用 Write 工具创建，内容遵循 [项目约定]：

**`<Name>.tsx` 模板**：
```[框架对应的代码块语言]
[根据技术栈生成的最小可用代码，含：导入、类型定义、组件主体、默认导出]
```

**`<Name>.test.tsx` 模板**：
```[测试框架对应的代码块]
[最小测试 skeleton，含：渲染断言、props 测试]
```

### 步骤 4：输出确认

输出形如：
```
✅ 已生成 <Name>
  ├── src/components/<Name>/<Name>.tsx
  ├── src/components/<Name>/<Name>.test.tsx
  ├── src/components/<Name>/<Name>.stories.tsx
  └── src/components/<Name>/index.ts

下一步：
  • 运行 `pnpm test <Name>` 验证测试通过
  • 在 [入口] 中导入：import { <Name> } from '@/components/<Name>'
```

---

## 严格遵守的约定

[直接来自 Q4 的项目约定 + 反模式]

- **命名规则**：[PascalCase 文件 + camelCase 变量 / kebab-case / ...]
- **导出方式**：[named export 优先 / default export 禁用 / ...]
- **禁止模式**：
  - 不用 `any`（必须显式类型或 `unknown`）
  - 不用 inline style（必须用 [Tailwind/CSS Modules]）
  - 不用 default export（除非框架要求，如 Next.js page）
  - [来自用户访谈的其他禁忌]

---

## 满分作业参考（生成质量标杆）

```[语言]
[来自 Q4 的标准实现完整代码，或符合该项目风格的示范]
```

---

## 遇到问题时

| 情况 | 处理方式 |
|------|---------|
| 参数缺失/不规范 | 报错指出哪里不对 + 给出正确用法示例 |
| 目标路径已存在 | [按 Q3 决策：报错 / 加序号 / 询问] |
| 父目录不存在 | 自动创建（`mkdir -p`），告知用户路径变化 |
| 项目根目录看不出来（无 package.json）| 报错：当前不在项目根目录，请 cd 到项目根再试 |
| 用户要求生成范围外的东西 | 礼貌说明边界，建议合适工具 |

---

## 自测清单

- [ ] **标准生成**：典型参数 → 生成全部文件 → `tsc --noEmit` 通过 → `eslint` 0 error
- [ ] **测试可跑**：生成的 `.test.tsx` 直接 `vitest run` 应该通过
- [ ] **参数校验**：缺参/错误命名应被拒绝并给出提示
- [ ] **路径冲突**：已存在的目标处理符合约定
```
