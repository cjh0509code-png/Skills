---
name: Skill 创建向导
emoji: 🏗️
category: AI 工具
skill: skill-creator
description: 面向非技术用户的顾问式访谈向导，4 个问题打包生成可直接安装的 AI 专属助手 Skill。
---

# skill-creator

面向完全不懂技术的业务人员的顾问式 AI 技能创建向导。

## 功能说明

Skill-Creator 通过 4 个问题的聊天式访谈，了解用户的工作需求，然后自动生成一个可直接安装到 Claude Code 的 `SKILL.md` 文件——用户无需了解任何代码或提示词工程知识。

所有技术环节（自由度评估、架构分流、边界与 Fallback 设计、测试矩阵生成）都在后台自动完成，用户只看到大白话提问和正式交付前的实际演示效果。

**核心能力：**
- **引导式访谈** — 4 个聚焦问题，每次只问一个，像顾问聊天一样推进
- **自由度评估** — 自动将任务分类为低/中/高自由度，分流到对应架构（规则型、模板型、原则型）
- **边界与 Fallback 设计** — 每个生成的技能内置 5 类边界场景处理
- **沙盒演练** — Claude 化身新训练的助手，在正式交付前先演示给用户看
- **测试矩阵** — 每个生成的技能内嵌 4 类验证场景（标准/边界/拒绝/压力）

**触发词：** `/skill-creator`、"帮我做个AI"、"创建技能"、"定制AI助手"、"我有个重复的工作"、"每次都要手动太烦了"、"有没有办法让AI帮我固定做这件事"、"打造专属AI助手"

## 文件结构

```
skill-creator/
├── SKILL.md                    ← 主控文件（精简版，<300 行）
├── references/
│   ├── template-low.md         ← 低自由度（精密型）输出模板
│   ├── template-medium.md      ← 中自由度（均衡型）输出模板
│   └── template-high.md        ← 高自由度（创意型）输出模板
└── README.md                   ← 本文件
```

## 安装

```bash
npx skills add <owner/repo> --skill skill-creator
```

## 参考文档

- `references/template-low.md` — 规则型任务的完整 SKILL.md 模板（数据转换、合规审核、格式转换）
- `references/template-medium.md` — 结构化写作任务的完整模板（邮件、报告、会议纪要）
- `references/template-high.md` — 创意与分析任务的完整模板（策略、文案、头脑风暴）
