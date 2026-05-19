# Skills 仓库规范

## 这个仓库是什么

个人博客的 Skill 展示仓库。博客会**直接读取本仓库内容**渲染展示卡片，
结构或格式不对会导致 Skill 无法显示或展示异常。

---

## 目录结构规则

每个 Skill 必须是根目录下的**独立文件夹**，文件夹内必须有 `README.md`：

```
Skills/                      ← 仓库根目录
├── image-gen-skill/
│   ├── README.md            ← 必须有，博客读取此文件
│   └── SKILL.md             ← Claude Code 安装用
├── skill-creator/
│   ├── README.md
│   ├── SKILL.md
│   └── references/
└── CLAUDE.md                ← 本文件，不会被读取为 Skill
```

**禁止**：
- 把 README.md 直接放根目录（会被忽略，不识别为 Skill）
- 用单个 `.md` 文件代替文件夹（必须是"文件夹 + README.md"）

---

## README.md 格式

每个 Skill 的 `README.md` 顶部应有 frontmatter，字段如下：

```markdown
---
name: 展示名称（中文，如：图片生成工具）
emoji: 🎨
category: 分类（如：AI 工具 / 开发效率 / 写作助手）
skill: 文件夹名（如：image-gen-skill，必须与文件夹名一致）
description: 一句话摘要，显示在博客卡片上，建议 20-40 字
---

## 功能说明

（正文内容）

## 安装

\`\`\`bash
npx skills add cjh0509code-png/Skills --skill <skill字段值>
\`\`\`
```

### 字段规则

| 字段 | 说明 | 规范 |
|------|------|------|
| `name` | 博客卡片展示名 | 中文，简洁，不超过 15 字 |
| `emoji` | 卡片图标 | **只用 1 个** emoji，多个会显示异常 |
| `category` | 分组标签 | 与现有 Skill 保持一致的命名风格 |
| `skill` | 安装时的 `--skill` 参数 | **必须与文件夹名完全一致**，只含小写字母和连字符，无空格 |
| `description` | 卡片摘要 | 建议填写，空着会导致卡片内容缺失 |

---

## 新增 Skill 的完整步骤

1. 在根目录创建文件夹，名称格式：`全小写-连字符`（如 `code-reviewer`）
2. 在文件夹内创建 `README.md`，顶部写 frontmatter（参考上方格式）
3. 在文件夹内创建 `SKILL.md`（Claude Code 安装使用的主文件）
4. 如有辅助文件（模板、引用文档），放在 `references/` 子目录
5. 提交并推送

---

## 提交信息规范

遵循 [Conventional Commits](https://www.conventionalcommits.org/)：

```bash
feat: add <skill-name> skill          # 新增 Skill
fix: fix <skill-name> description     # 修复问题
docs: update <skill-name> readme      # 更新文档
refactor: restructure <skill-name>    # 重构
```

---

## 常见错误

| 错误 | 正确做法 |
|------|---------|
| `skill: image gen skill`（含空格） | `skill: image-gen-skill` |
| `emoji: 🎨🖼️`（多个 emoji） | `emoji: 🎨`（只用一个） |
| README.md 放在根目录 | 必须放在各自的子文件夹内 |
| 文件夹名与 `skill` 字段不一致 | 两者必须完全相同 |
| frontmatter 缩进错误（YAML 语法） | 用标准 YAML，key 后面跟 `: ` |

---

## 当前已有 Skill 的 category 参考

| Skill | category |
|-------|---------|
| image-gen-skill | AI 工具 |
| skill-creator | AI 工具 |
