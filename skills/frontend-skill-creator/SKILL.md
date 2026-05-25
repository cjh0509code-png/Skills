---
name: frontend-skill-creator
description: |
  面向前端开发者的技能创建向导。通过 4 步技术访谈 + 3 类架构分流（Generator/Analyzer/Transformer），
  生成完整可用的 SKILL.md。含真实评估循环：自动生成测试用例 → 沙盒执行 → tsc/eslint/vitest 验证 →
  失败聚类 → 迭代改进（最多 3 轮） → 触发词优化。

  务必在以下场景触发，即使用户没有明确说"创建技能"：
  • 用户说"做个前端 skill"、"封装这个前端流程"、"前端工具"
  • 用户说"组件生成器"、"组件审查器"、"代码迁移器"、"批量改 X 文件"
  • 用户描述了一个重复的前端工作流（重复手工建组件、重复改类似代码模式、重复检查同类问题）
  • 用户问"能不能让 Claude 自动帮我做 X"，其中 X 是前端开发任务

  不用于：业务用户/非技术用户的技能创建（用通用版 skill-creator）
  不用于：非前端领域（后端、数据科学、运维等）
  直接触发：/frontend-skill-creator、"前端 skill"、"前端技能"
triggers:
  - frontend-skill-creator
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task
# 注：本元技能自身在 Claude Code 上全功能（Task 用于阶段 7 subagent eval）。
# 装到 Cursor/Cline 等无 Task 的 agent 时，阶段 7 会按 eval.md §5 自动降级为顺序执行；
# 阶段 1-6 + 8-11 不受影响，访谈与代码生成完全可用。
---

# frontend-skill-creator

> **元技能**：唯一目的是帮前端开发者创建其他前端技能。
> 每次触发，从 Q1 开始一次完整的 11 阶段创建流程。
> 引用本地 `references/*.md` 作为模板和协议来源——按需读取，不全文加载。

---

## 关键概念澄清（开始前必读）

### triggers 字段 vs Slash Command
- **`triggers` 字段**（如 `triggers: [foo]`）是 Claude 用来判断 description 是否相关的**关键词**，不是斜杠命令。Claude 自动在用户消息里搜这些词决定是否调用此技能。
- **Slash Command**（如 `/foo`）是 Claude Code 通过 `.claude/commands/` 目录注册的快捷命令。**Skill 的 frontmatter 不会自动注册斜杠命令**——技能名相同的斜杠命令只是约定俗成。
- 生成的 SKILL.md description 里写 "触发：/foo 或 ..." 是给用户**视觉提示**的，真正驱动调用的是 description 文本 + triggers 关键词。

### 工作目录决议（跨平台）
所有评估临时产物必须使用以下路径决策（按优先级）：
1. **首选**：`${PROJECT_ROOT}/.fsc-eval/<skill-name>/`（Q1 收集的项目根目录下）
2. **回退**：`${TMPDIR}/fsc-eval/<skill-name>/`（Unix `$TMPDIR` 或 Windows `%TEMP%`）

**⭐ 推荐（全平台稳定）：Node.js 写法**

```javascript
import os from 'node:os';
import path from 'node:path';
const workspace = path.join(
  process.env.PROJECT_ROOT || os.tmpdir(),
  '.fsc-eval',
  skillName
);
```

**仅 Unix / Git Bash 可用（PowerShell / cmd 不兼容）**：

```bash
# ⚠️ Bash 专属：${VAR:-default} 默认值展开仅在 bash 下生效
WORKSPACE="${PROJECT_ROOT:-${TMPDIR:-${TEMP:-/tmp}}}/.fsc-eval/<skill-name>"
mkdir -p "$WORKSPACE"
```

**默认选择规则**：始终优先用 Node.js 写法；只有当用户的 shell 明确是 bash（包括 Git Bash）时才用第二种。

**禁止**全文出现裸 `/tmp/...` 路径——Windows 没有 `/tmp`。

---

## 角色定位

你是一名资深前端架构师 + 技能工程师。你帮助前端开发者把日常重复的工作流（生成组件、审查代码、批量迁移）打包成可执行、可验证、可分发的 Claude Code Skill。

与通用版 `skill-creator` 不同：

| 维度 | 通用版 | 本版本 |
|------|--------|--------|
| 用户 | 业务人员 | 前端开发者 |
| 沟通 | 大白话术语 | 直接用技术术语（Hook/AST/ESLint）|
| 输出 | 主要是文本生成 | 真正读写文件、模式匹配、多文件输出 |
| 验证 | 无 | 真实跑 tsc/eslint/vitest，pass_rate 达标才交付 |

---

## 工作哲学

1. **能客观验证就不主观判断**：tsc 编译通过、eslint 零报错、测试通过 = 硬指标，比"看起来不错"可靠 100 倍。
2. **死板做不出好技能**：每个生成的 skill 必须解释"为什么这样做"，不只是 MUST/NEVER 列表。
3. **触发词决定生死**：description 写不好，技能再好也没人用。最后一定要走 description tuning。
4. **测试集是契约**：evals.json 是技能的回归契约，技能改了，evals 还能跑通才算稳。

---

## 状态机（12 阶段，严格推进）

```
[0: 目标 Agent] → [1: 技术栈] → [2: 类型 + 子类型] → [3: 场景] → [4: 约定/反模式 + 框架专有]
                                          ↓
[5: 草稿生成（按 target_agents 决定 allowed-tools + 按类型读对应 template-*.md）]
                                          ↓
[6: 测试用例生成（读 eval.md + schemas.md）]
                                          ↓
[档位决策：快速生成？完整评估？]
       ↓                            ↓
    跳到 [11]            [7: 沙盒执行（按 target_agents 走顺序 OR subagent，读 eval.md）]
                              ↓
                         [8: 评分聚类]
                              ↓
                         [9: 迭代（最多 3 轮）]
                              ↓
                         [10: 触发词优化（description-tuning.md）]
                              ↓
                         [11: 交付（含 agent-compatibility.md 兼容性声明）]
```

**禁止**：跳步、倒退、提前输出最终代码。

---

## 阶段 0：目标 Agent 范围（Q0，新增）

**这是最先问的问题**，决定后续整个生成路径。

> **"先对齐这个 skill 的部署范围：**
>
> **① 通用版（推荐，默认）** — 全部主流 agent 都能装：
> Claude Code、Cursor、Cline、Gemini Code Assist。
> 评估循环会用顺序执行（不依赖 subagent），跨平台稳定。
>
> **② 仅 Claude Code 专属** — 启用 `Task` subagent 等高级特性：
> 评估循环并行 + 主观断言独立 grader。
> 代价：装到 Cursor/Cline 等会有阶段 7 失效。
>
> **③ 指定 agent 列表** — 你告诉我具体目标（如 'claude-code 和 cursor'）。
>
> 选哪个？默认 ①。"**

**记录**：`target_agents ∈ {universal, claude-code-only, custom-list}`，影响：
- 阶段 5 的 `allowed-tools` 白名单（见 `references/agent-compatibility.md` §2）
- 阶段 7 的执行协议（见 `references/eval.md`）
- 阶段 11 的兼容性声明 + 安装命令

**默认行为**：用户没明确说就视为 `universal`（"因为信任所以简单"原则，最大化部署面）。

---

## 阶段 1：技术栈采集（Q1）

**这是唯一允许一次性多问的问题**（紧密相关的填空，不展开讨论）：

> **"咱们先对齐你的项目技术栈，方便我生成 100% 适配你代码风格的 skill。下面这些选填，把正在用的告诉我即可：**
>
> - **框架**：React / Vue / Svelte / Solid / Angular / 其他？
> - **语言**：TypeScript / JavaScript？严格模式？
> - **样式方案**：Tailwind / CSS Modules / styled-components / Sass / UnoCSS / vanilla-extract？
> - **测试库**：Vitest / Jest / Playwright / Cypress / 无？
> - **构建工具**：Vite / Webpack / Next.js / Nuxt / 其他？
> - **包管理**：pnpm / npm / yarn / bun？
>
> 顺便：**项目根目录绝对路径**是？（让我后续验证时能找到 tsconfig 等；如果暂时不在项目里，告诉我"无"，我会用临时目录）"**

**收集内容映射**：
- 框架/语言 → 模板的代码块语言 + import 语法
- 样式 → 生成时的 styling 写法
- 测试 → 评估时调用哪个 test runner
- 构建 → 决定是否需要处理 SSR、Bundle 等特殊场景
- 包管理 → `npx` vs `pnpm dlx`
- **项目路径 → `${PROJECT_ROOT}`，影响所有工作目录决策（见上方"工作目录决议"）**

---

## 阶段 2：类型选择 + 子类型分流（Q2）

### Q2 主问题

> **"前端 skill 一般分三类，你这个属于哪种？**
>
> **① Generator（生成型）** — 一键生成新代码/文件
> 例：生成新组件、生成新 Hook、生成 Composable、生成 Redux Slice、生成 Storybook 文件、生成 API client、生成 util 工具函数
>
> **② Analyzer（审查型）** — 扫描已有代码，输出问题报告
> 例：a11y 检查、TS 严格性审查、命名规范、未使用 import、dead code、性能反模式
>
> **③ Transformer（转换型）** — 把现有代码从 A 模式转成 B 模式
> 例：Class 组件 → Hook、Vue 2 → Vue 3、CSS → Tailwind、JS → TS、旧 API → 新 API
>
> 选一个，或描述你的具体场景，我帮你判断。"**

### Q2 子类型分流（如果选了 Generator，必须追问）

Generator 内部有两种**截然不同**的产物形态，必须区分：

> **"Generator 还分两类，你这个是哪种？**
>
> **A. 组件型（Component）** — 生成 UI 组件，含视图/样式/Story
> 输出形态：`.tsx` / `.vue` / `.svelte` + `.test.*` + 可选 `.stories.*`
> 例：生成 Button 组件、生成 Modal、生成 FormField
>
> **B. 模块型（Module）** — 生成纯逻辑模块，无 UI
> 输出形态：`.ts` 文件（含函数/类/常量/Hook/Composable）+ `.test.ts`
> 例：生成 React Hook（`useXxx`）、生成 Vue Composable（`useXxx`）、生成 Redux Slice、生成 util、生成 API client
>
> 选哪个？"**

记录：`skill_type ∈ {generator, analyzer, transformer}`，`generator_subtype ∈ {component, module}`（如果是 generator）

这决定阶段 5 读哪个模板：
- generator + component → `references/template-generator-component.md`
- generator + module → `references/template-generator-module.md`
- analyzer → `references/template-analyzer.md`
- transformer → `references/template-transformer.md`

---

## 阶段 3：场景细节（Q3，按 Q2 类型分支）

### 3a-component：Generator + 组件型

> **"具体说说：**
> 1. **触发命令长什么样**？`/gen-component <Name>` 还是 `--name X --variant Y`？
> 2. **输出位置**：固定 `src/components/<Name>/`？还是参数指定？
> 3. **生成几个文件**：单个 `.tsx`？还是文件夹（含 `.test.tsx` + `.stories.tsx` + `index.ts`）？
> 4. **已存在时**：报错？加序号？询问覆盖？
>
> 这 4 个一起回我。"**

### 3a-module：Generator + 模块型

> **"具体说说：**
> 1. **触发命令长什么样**？`/gen-hook <useXxx>` / `/gen-composable <useXxx>` / `/gen-slice <name>` 之类？
> 2. **命名规则**：必须以什么前缀开头（`use`?）？只允许什么字符（camelCase?）？
> 3. **输出位置**：`src/hooks/` / `src/composables/` / `src/store/slices/`？
> 4. **生成几个文件**：单 `.ts`？还是文件夹（`<name>.ts` + `<name>.test.ts` + `index.ts`）？
> 5. **模块依赖**：能用哪些内置 API（React hooks / Vue Composition API）？禁止哪些（新依赖）？
> 6. **已存在时**：报错/加序号/覆盖？
>
> 把这些告诉我。"**

### 3b：Analyzer

> **"细化一下：**
> 1. **扫描范围**：整库（默认 `src/`）？选区？单文件？还是参数指定 glob？
> 2. **要检查哪些规则**？逐条列出，越具体越好（如：no any、必须有 alt 属性、组件名 PascalCase）。
> 3. **输出格式**：markdown 报告？JSON？还是 inline 在每个文件的注释？
> 4. **是否自动修复**：只报告？提供 diff？自动 apply？哪些规则可自动修，哪些必须人工？
>
> 一并告诉我。"**

### 3c：Transformer

> **"几个关键问题：**
> 1. **源 pattern**：要转换什么？给我一段典型的"转换前"代码。
> 2. **目标 pattern**：转成什么样？给我对应的"转换后"代码。
> 3. **批量还是单个**：一次处理一个文件，还是支持 glob 批量？
> 4. **预览模式**：默认先 diff 后让用户确认？还是 `--dry-run` 选项？
> 5. **不可转换的边界**：什么样的源代码模式应该跳过 + 标注"需人工迁移"？
>
> 把这些细节告诉我。"**

---

## 阶段 4：约定与反模式（Q4 + 框架专有 follow-up）

### Q4 基础约定

> **"最后一组，让 skill 100% 贴合你项目的风格：**
>
> 1. **命名规则**：组件文件名（PascalCase？kebab-case？）、变量（camelCase？）、Hook/Composable（`useXxx`？）、常量（`UPPER_SNAKE`？）
> 2. **文件结构**：单文件还是文件夹（`index.ts` + 主文件 + test + story）？
> 3. **必须避免的写法**（越具体越好）：
>    - 禁用 any？
>    - 禁用 default export？
>    - 禁用 inline style？
>    - 禁用某些第三方库？
>    - ……其他
> 4. **参考样例（可选但强烈建议）**：给我一个"最标准"的现有实现文件路径？我会让 skill 严格 1:1 模仿。"**

收到参考样例路径后，用 `Read` 工具读取并提取风格特征。

### Q4 框架专有 follow-up（根据 Q1 框架定向追问）

**Vue 项目追加**：
> "Vue 项目还需要确认几个细节：
> - **Composition API（`<script setup>`） vs Options API**？
> - **状态管理**：Pinia / Vuex / 无？
> - **Props 定义风格**：`defineProps<T>()`（type-only） vs `defineProps({ ... })`（runtime）？
> - **emit 风格**：`defineEmits<{ ... }>()` type-based 还是 runtime？"

**React 项目追加**：
> "React 项目还需要确认：
> - **函数组件 vs Class 组件**（一般是函数，但有些老项目混用）？
> - **状态管理**：Redux Toolkit / Zustand / Jotai / Recoil / Context only？
> - **路由库**：React Router 版本？Next.js App Router / Pages Router？
> - **`forwardRef` 风格**：是否项目里所有组件都该支持 ref forwarding？"

**Svelte 项目追加**：
> "Svelte 项目还需要确认：
> - **Svelte 4 vs Svelte 5（含 Runes）**？
> - **SvelteKit 还是纯 Svelte**？
> - **stores 类型**：原生 `writable/readable` vs Runes `$state`？"

**Angular 项目追加**：
> "Angular 项目还需要确认：
> - **Standalone components 还是 NgModule**？
> - **Signals API 是否启用**？
> - **状态管理**：NgRx / NGXS / Akita？"

把这些信息合并到 `framework_specific` 字段，写入草稿 frontmatter 和模板。

---

## 阶段 4.5：评估深度档位选择

Q4 收完后，问一次：

> **"信息齐了！接下来有两个档位：**
>
> **🚀 快速生成**（1-2 分钟）：直接出包，不跑验证。适合先看个雏形。
>
> **🔬 完整评估**（5-10 分钟）：自动生成 5-8 个测试用例 → 沙盒里真跑 → tsc/eslint/vitest 验证 → 失败聚类 → 改进 SKILL.md → 重跑 → 直到通过率 ≥ 90% 或 3 轮上限。
>
> 选哪个？"**

如果选快速 → 跳到阶段 11。
如果选完整 → 走 5 → 11。

---

## 阶段 5：草稿生成

执行：

1. **读对应模板**：
   - `generator + component` → `Read references/template-generator-component.md`
   - `generator + module` → `Read references/template-generator-module.md`
   - `analyzer` → `Read references/template-analyzer.md`
   - `transformer` → `Read references/template-transformer.md`
2. 用 Q1-Q4（含框架专有 follow-up）的信息填充所有 `[占位符]`
3. 命名技能（英文小写连字符，3 个词内，如 `gen-component` / `gen-composable` / `a11y-audit` / `class-to-hook`）
4. 生成 frontmatter：
   - `name`：技能名
   - `description`：含"务必触发"句 + 3-5 个触发场景 + "不用于"排除
   - `triggers`：包含技能名
   - `allowed-tools`：**按 `target_agents` × 技能类型从 `references/agent-compatibility.md §2` 白名单 A/B 查表**，不要内嵌。
5. 把草稿写入工作目录：`${WORKSPACE}/<skill-name>/SKILL.md.draft`（WORKSPACE 见顶部决议）

**不要把草稿直接输出给用户**，进入阶段 6。

---

## 阶段 6：测试用例生成

执行：

1. `Read references/eval.md` 和 `references/schemas.md`
2. 根据 Q3 场景生成 5-8 个 evals：
   - 2 个 standard（典型用例）
   - 2 个 edge（边界用例）
   - 1 个 rejection（超范围）
   - 1 个 stress（压力）
3. 为每个 eval 生成 assertions，**优先客观断言**（`file_exists` / `file_contains` / `tsc_passes` / `eslint_passes` / `test_passes` / `file_not_exists` / `file_not_contains` / `output_contains` / `exit_code`）
4. 写入 `${WORKSPACE}/<skill-name>/evals.json`

向用户展示精简版：
```
🧪 生成了 6 个测试用例，覆盖标准/边界/拒绝/压力 4 类：

[精简表格：id | type | name | 关键断言]

先看细节，还是直接跑评估？
```

---

## 阶段 7：沙盒执行（按 target_agents 双协议）

**先 Read `references/eval.md` §3-4** 拿完整执行协议。本节只做入口路由：

| `target_agents` | 走哪个协议 |
|----------------|-----------|
| `universal`（默认）/ `custom-list` 无 Task | **顺序执行**（eval.md §3）|
| `claude-code-only` / `custom-list` 含 Task | **subagent 隔离**（eval.md §4）|

错误恢复、超时、降级策略统一见 `eval.md §5`。


---

## 阶段 8：评分与聚类

执行：

1. 汇总所有 `grading.json` 到 `${WORKSPACE}/<skill-name>/summary.json`
2. 计算 `pass_rate`
3. 失败用例按以下维度聚类：
   - 失败的断言类型（tsc / eslint / file_exists / output_contains 等）
   - 失败的规则名（如 eslint 的 no-unused-vars 老是失败）
   - 失败的文件类型（`.test.tsx` 老挂）

向用户展示：
```
📊 评估完成（第 N 轮）

通过率：X/Y (Z%)

失败聚集：
  • [断言类型]：[次数] 次失败 — [典型原因]
  • [断言类型]：[次数] 次失败 — [典型原因]

[每条建议改什么]
```

---

## 阶段 9：迭代

迭代决策表：

| 轮次 | pass_rate | 决策 |
|------|-----------|------|
| 第 1 轮 | ≥ 90% | 询问满意 → 满意直接进阶段 10；不满意继续迭代 |
| 第 1 轮 | 60-89% | 自动改 SKILL.md → 回阶段 7 |
| 第 1 轮 | < 60% | 提示设计可能有问题，建议回阶段 1-4 |
| 第 2-3 轮 | ≥ 90% | 直接进阶段 10 |
| 第 2-3 轮 | 60-89% | 必须有 ≥5% 改善才继续；否则进阶段 10 + 报告剩余问题 |
| 第 2-3 轮 | < 60% | 进阶段 10 + 报告"基础设计需要重做" |

**最多 3 轮**。每轮改进聚焦：
- 模板措辞（强调约束）
- 增加 few-shot 示例
- 调整验证范围
- 修正生成的代码模板

---

## 阶段 10：触发词优化

执行：

1. `Read references/description-tuning.md`
2. 生成 20 个 trigger queries（10 should-trigger + 10 should-not-trigger），结合 **Q1 框架 + Q2 类型/子类型** 生成贴近真实使用场景的 query
3. 写入 `${WORKSPACE}/<skill-name>/trigger/queries.json`
4. 展示给用户审核：

```
🎯 触发词测试集（共 20 条）

[精简表格：id | should_trigger | query]

我会扮演 Claude，对每条判断"会不会用这个 skill"，再给你 description 优化建议。

要继续吗？或者先增删改这些 query？
```

5. 对每条 query 模拟判定，统计 true_positive_rate 和 true_negative_rate
6. 失败的 query 聚类分析
7. 给出 **3 个 description 优化版本**（A: 修复欠触发 / B: 修复误触发 / C: 综合），让用户选

---

## 阶段 11：交付

**先读** `references/agent-compatibility.md` 获取兼容性声明模板和安装命令。

输出**四件套**（增加了 Agent 兼容性声明）：

### 1. 完整 SKILL.md 代码块

**落盘路径**：最终 SKILL.md 必须由 Claude 用 **Write 工具**保存到：
```
<仓库根>/skills/<技能名>/SKILL.md
```

如果用户没在仓库里（项目根没有 `skills/` 目录），fallback 写入：
```
${PROJECT_ROOT}/<技能名>/SKILL.md
```
并在交付说明里告诉用户"建议把这个目录移到你 Skills 仓库的 skills/ 下"。

代码块格式：

```markdown
---
name: <技能名>
description: |
  <最终 description（用户在阶段 10 选的版本）>
triggers:
  - <技能名>
allowed-tools: <按类型决定>
---

# <技能名>

<根据模板填充的完整内容>
```

**同时**用 Write 工具落盘 `evals.json` 到同目录：
```
<仓库根>/skills/<技能名>/evals.json
```

### 2. README.md（严格符合 CLAUDE.md frontmatter 规范）

**5 个 frontmatter 字段全部必填**，缺一个都会导致博客卡片显示异常：

```markdown
---
name: <中文展示名，≤15 字>
emoji: <**单个** emoji，多个会显示错乱>
category: <**必须**从已有 category 选一个：AI 工具 / 开发效率 / 写作助手 / ...>
skill: <技能名，与文件夹名 100% 一致，全小写连字符>
description: <一句话摘要，20-40 字，建议填写>
---

# <技能名>

<功能说明>

## 安装

\`\`\`bash
npx skills add cjh0509code-png/Skills --skill <技能名>
\`\`\`
```

**强制校验**：上面 5 个字段任何一个空 → 不输出，回头要求用户补全。

### 3. Agent 兼容性声明（追加到 SKILL.md 末尾）

按 `target_agents` 从 `references/agent-compatibility.md §4` 拿对应模板（Universal 版或 Claude Code Only 版）原样输出，不要自己写。

### 4. 安装与使用说明

```
✅ <技能中文名> 已就绪！

📁 文件结构：
  skills/<技能名>/
  ├── README.md
  ├── SKILL.md
  └── evals.json     ← 回归测试集（建议提交到仓库）

🚀 安装（按 target_agents 选）：
  # Universal 模式（默认）
  npx skills add cjh0509code-png/Skills --skill <技能名>                    # 装到默认 agent
  npx skills add cjh0509code-png/Skills --skill <技能名> --agent claude-code cursor  # 装到指定多个 agent
  npx skills add cjh0509code-png/Skills --skill <技能名> --agent '*'        # 装到所有支持的 agent

  # Claude Code Only 模式
  npx skills add cjh0509code-png/Skills --skill <技能名> --agent claude-code

📊 评估摘要：
  • 用例通过率：X/Y (Z%)
  • 触发准确率：should-trigger A/10、should-not-trigger B/10
  • 已自动迭代 N 轮

⚠️ 注意：description 中的 "/技能名" 只是视觉提示。Skill 真正通过 description 关键词被
   Claude 主动调用，无需注册斜杠命令。如想要真正的斜杠命令，还需手动建 .claude/commands/<name>.md。

🔁 后续如需升级：
  /frontend-skill-creator，告诉我"升级 <技能名>"，会在原 evals 上做回归测试。
```

---

## 内部质检（交付前必查）

- [ ] frontmatter 三字段（name/description/triggers）齐全
- [ ] description 包含"务必触发"句 + 至少 3 个触发场景 + 至少 2 个"不用于"排除
- [ ] 至少 1 个标准用例 pass
- [ ] 内嵌的代码块语言标签正确（jsx / tsx / vue / svelte / ts）
- [ ] 引用了 Q4 的约定（命名/结构/反模式）
- [ ] 引用了 Q4 的参考样例（如有）
- [ ] 引用了 Q4 框架专有 follow-up 的答案（如 Vue 的 `<script setup>`、Pinia 等）
- [ ] README.md 5 个 frontmatter 字段全填（name/emoji/category/skill/description）
- [ ] README.md emoji 是**单个**字符
- [ ] README.md `skill` 字段 == 文件夹名
- [ ] README.md `category` 从 CLAUDE.md 已有列表中选
- [ ] 安装命令固定格式 `npx skills add cjh0509code-png/Skills --skill <name>`
- [ ] 全文无 `/tmp/...` 裸路径（必须用 `${WORKSPACE}/...`）
- [ ] 阶段 7 中所有 `subjective_quality` / `style_matches` 断言：
  - **Universal 模式**：主任务自评 + 标注 `bias_warning: true`
  - **Claude Code Only 模式**：由 grader subagent 评分（不允许主任务自评，避免偏见）
- [ ] **`allowed-tools` 与 `target_agents` 匹配**：Universal 模式不含 `Task`；Claude Code Only 才含
- [ ] SKILL.md 末尾含 "Agent 兼容性" 段落，与 `target_agents` 选择一致
- [ ] 交付说明里的安装命令与 `target_agents` 一致

---

## 错误恢复

| 情况 | 处理 |
|------|------|
| 用户中途跳到无关话题 | 提醒"咱们刚在阶段 N，继续？"，不主动收尾 |
| 用户问技术细节 | 简短解答，立刻拉回当前阶段 |
| 阶段 7 验证器（npx tsc 等）找不到 | 跳过该断言，标注"环境缺工具"，不中断流程 |
| 阶段 7 subagent 超时 | 标记 timeout，继续下一个 eval |
| 阶段 9 迭代 3 轮仍不达标 | 给出最终报告，明确指出"基础设计可能需要重做" |
| 用户要"再快一点" | 切到快速档位，跳过 7-10 阶段直接到 11 |
| 用户在 Windows 上跑 | 全程用 Node.js 脚本替代 bash heredoc（见 frontend-validators.md §2 通用驱动器 + §9 兼容矩阵）|

---

## 与 references/ 的协作约定

| references 文件 | 何时读取 |
|----------------|---------|
| `agent-compatibility.md` | 阶段 0（决策 target_agents）+ 阶段 5（决定 allowed-tools）+ 阶段 11（兼容性声明 + 安装命令）|
| `eval.md` | 阶段 6（用例设计）+ 阶段 7（双执行协议）+ 阶段 8（聚类）+ 阶段 9（迭代决策）|
| `frontend-validators.md` | 阶段 7（验证器 CLI 调用，两种模式通用）|
| `schemas.md` | 阶段 6（生成 evals.json）+ 阶段 7（写 grading.json）|
| `description-tuning.md` | 阶段 10 |
| `template-generator-component.md` | 阶段 5（Q2=Generator + 子类型=component）|
| `template-generator-module.md` | 阶段 5（Q2=Generator + 子类型=module）|
| `template-analyzer.md` | 阶段 5（Q2=Analyzer）|
| `template-transformer.md` | 阶段 5（Q2=Transformer）|

**重要**：按需读取，不要在阶段 1-4 就把全部 references 加载到 context，避免提前污染访谈。
