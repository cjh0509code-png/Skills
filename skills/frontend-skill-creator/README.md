---
name: 前端技能创建向导
emoji: ⚛️
category: AI 工具
skill: frontend-skill-creator
description: 默认生成跨 agent 通用前端 Skill，按生成/审查/重构分流，含真实评估循环和触发词优化。
---

# frontend-skill-creator

面向前端开发者的专属技能创建向导。两大差异化能力：

1. **默认跨 agent 通用** — 生成的子技能在 Claude Code / Cursor / Cline / Gemini Code Assist 都能装、都能跑（除非你明确要 Claude Code 专属）
2. **含完整评估循环** — 真正生成代码、跑 TypeScript / ESLint / Vitest 验证，迭代到通过率达标才交付

## 功能说明

| 阶段 | 做什么 |
|------|--------|
| **0** | **目标 agent 范围**（Universal / Claude Code Only / Custom List）|
| 1-4 | 4 步技术访谈（技术栈 → 类型 → 场景 → 约定/反模式）|
| 5 | 草稿生成（按 target_agents 决定 allowed-tools + 按 Generator/Analyzer/Transformer 加载模板）|
| 6 | 测试用例生成（5-8 个 evals + 客观断言）|
| 7 | 沙盒演练（Universal 顺序执行 / Claude Code 用 Task subagent 并行 + tsc/eslint/vitest 自动验证）|
| 8 | 评分与失败聚类 |
| 9 | 用户 review + 迭代（最多 3 轮）|
| 10 | 触发词优化（should/should-not-trigger 测试集）|
| 11 | 交付完整 SKILL.md + README + 安装命令 |

### 三类技能架构分流

- **Generator** — 生成新组件 / Hook / Slice / Storybook / API client
- **Analyzer** — a11y 检查 / TS 严格性审查 / 命名规范 / dead code 扫描
- **Transformer** — Class→Hook / Vue2→Vue3 / CSS→Tailwind / JS→TS 迁移

### 评估深度档位

访谈结束后可选：
- **快速生成** — 只走 1-5 + 11 阶段，几分钟出包
- **完整评估** — 走完 11 阶段，含真实验证 + 迭代，质量更可靠

## 触发词

`/frontend-skill-creator`、"前端 skill"、"做个前端技能"、"封装这个前端流程"、"组件生成器"、"代码审查器"、"代码迁移工具"

## 安装

```bash
npx skills add cjh0509code-png/Skills --skill frontend-skill-creator
```

## 文件结构

```
frontend-skill-creator/
├── README.md
├── SKILL.md                          # 主控（含 12 阶段状态机）
└── references/
    ├── agent-compatibility.md          # Agent 能力矩阵 + 安装命令对照
    ├── eval.md                          # 评估协议（测试用例 + 双执行协议 + 迭代决策）
    ├── frontend-validators.md           # 前端验证集成（tsc/eslint/vitest，跨平台）
    ├── description-tuning.md            # 触发词优化流程
    ├── schemas.md                       # JSON 范例 + 12 种断言速查
    ├── template-generator-component.md  # 生成型-组件子模板
    ├── template-generator-module.md     # 生成型-模块子模板（Hook/Composable/Slice/Util）
    ├── template-analyzer.md             # 审查型模板
    └── template-transformer.md          # 转换型模板
```
