# Agent 兼容性矩阵

> 本文档定义 SKILL.md 协议在主流 AI agent 上的支持度，以及如何按 `target_agents` 决策生成不同的 `allowed-tools` 与执行协议。
> 主控 SKILL.md 在阶段 5（决定 allowed-tools）和阶段 11（输出兼容性声明）引用本文档。

---

## 1. 主流 Agent 能力矩阵

| Agent | 文件读 | 文件写 | 文件编辑 | Shell/Bash | 文件搜索 | 内容搜索 | 子代理 | SKILL.md 加载 | 推荐用途 |
|-------|------|------|--------|----------|--------|--------|------|--------------|---------|
| **Claude Code** | `Read` | `Write` | `Edit` | `Bash` | `Glob` | `Grep` | `Task` ✅ | ✅ 原生 | 全功能基线 |
| **Cursor** | ✅ 内置 | ✅ 内置 | ✅ Apply | ✅ 终端 | ✅ codebase | ✅ codebase search | ❌ | ✅ 通过 skills CLI | 全功能（少 subagent）|
| **Cline (VS Code)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ 通过 skills CLI | 同 Cursor |
| **Gemini Code Assist** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ 通过 skills CLI | 同 Cursor |
| **Claude.ai Skills** | ✅ 沙箱 | ✅ 沙箱 | ❌ | ⚠️ 受限 Python | ⚠️ | ⚠️ | ❌ | ✅ 原生 | 仅访谈 + 文本生成 |
| **ChatGPT Custom GPT** | ⚠️ Knowledge files | ❌ | ❌ | ✅ Python interp | ❌ | ❌ | ❌ | ❌ 需手动改造 | 仅对话流，需重写工具调用层 |

---

## 2. 目标 Agent 模式（Q0 输入 → 决策表）

| 用户选择 | 实际目标 | allowed-tools 白名单（按技能类型）| 阶段 7 执行协议 |
|---------|---------|--------------------------------|----------------|
| **Universal**（默认，推荐）| 所有支持 SKILL.md 的 agent | 见下方"白名单 A" | 顺序执行（见 eval.md §3）|
| **Claude Code Only** | 仅 claude-code | 见下方"白名单 B" | subagent 并行（见 eval.md §4）|
| **Custom List** | 用户指定（如 `claude-code,cursor`）| 按列表的**最小公共能力集**取交集 | 若交集含 `Task` → 并行；否则顺序 |

### 白名单 A：Universal 模式（lowest common denominator）

| 技能类型 | allowed-tools |
|---------|--------------|
| Generator | `Read, Write, Glob, Bash` |
| Analyzer | `Read, Grep, Glob, Bash` |
| Transformer | `Read, Edit, Write, Grep, Glob, Bash` |

**无 `Task`**——确保 Cursor/Cline 等无 subagent 的 agent 也能跑。

### 白名单 B：Claude Code Only 模式

| 技能类型 | allowed-tools |
|---------|--------------|
| Generator | `Read, Write, Glob, Bash, Task` |
| Analyzer | `Read, Grep, Glob, Bash, Task` |
| Transformer | `Read, Edit, Write, Grep, Glob, Bash, Task` |

---

## 3. 安装命令对照（输出到生成的 README）

按 `target_agents` 输出不同的安装段：

### Universal 模式

```bash
# 装到当前默认 agent
npx skills add cjh0509code-png/Skills --skill <name>

# 装到指定 agent（可多个）
npx skills add cjh0509code-png/Skills --skill <name> --agent claude-code cursor

# 装到所有支持的 agent
npx skills add cjh0509code-png/Skills --skill <name> --agent '*'
```

### Claude Code Only 模式

```bash
# 仅推荐安装到 Claude Code（其他 agent 部分阶段会失效）
npx skills add cjh0509code-png/Skills --skill <name> --agent claude-code
```

### Custom List 模式

```bash
# 用户指定的 agent 列表
npx skills add cjh0509code-png/Skills --skill <name> --agent <list>
```

---

## 4. 兼容性声明模板（嵌入生成的 SKILL.md 末尾）

### Universal 版本

```markdown
## Agent 兼容性

本技能面向所有支持 SKILL.md 协议的 agent，无平台专属依赖：

| Agent | 状态 |
|-------|------|
| Claude Code | ✅ 全功能 |
| Cursor | ✅ 全功能 |
| Cline (VS Code) | ✅ 全功能 |
| Gemini Code Assist | ✅ 全功能 |
| Claude.ai Skills | ⚠️ 仅文本生成阶段可用（沙箱无 Bash）|
| ChatGPT Custom GPT | ⚠️ 需自行适配工具调用层 |
```

### Claude Code Only 版本

```markdown
## Agent 兼容性

⚠️ **本技能为 Claude Code 专属版本**，使用了 `Task` subagent 等平台独有能力。

| Agent | 状态 |
|-------|------|
| Claude Code | ✅ 全功能 |
| 其他 agent | ❌ 阶段 7（评估循环）失效，因为无 `Task` subagent；建议改装 Universal 版本 |

如需跨平台版本，请联系作者重新生成 Universal 模式。
```

---

## 5. 工具名映射表（未来扩展）

不同 agent 的等价工具可能名称不同。Universal 模式当前**只用 SKILL.md 协议标准工具名**（`Read` / `Write` / `Edit` / `Bash` / `Grep` / `Glob`），各 agent 的 skills CLI 适配层会自动翻译。

| SKILL.md 标准工具 | Claude Code | Cursor | Cline | Gemini |
|------------------|------------|--------|-------|--------|
| `Read` | ✅ 原生 | ✅ 内置 | ✅ | ✅ |
| `Write` | ✅ | ✅ Apply | ✅ | ✅ |
| `Edit` | ✅ | ✅ Apply | ✅ | ✅ |
| `Bash` | ✅ | ✅ Terminal | ✅ | ✅ |
| `Glob` | ✅ | ✅ codebase | ✅ | ✅ |
| `Grep` | ✅ | ✅ codebase search | ✅ | ✅ |
| `Task` | ✅ subagent | ❌ | ❌ | ❌ |

---

## 6. 决策判断流程（Claude 在阶段 5 执行）

```
1. 读取 Q0 收集到的 target_agents
2. 若 target_agents == "universal":
     → allowed-tools 用白名单 A
     → 阶段 7 用顺序执行协议
     → README 含 Universal 兼容性声明
3. 若 target_agents == "claude-code-only":
     → allowed-tools 用白名单 B
     → 阶段 7 用 subagent 协议
     → README 含 Claude-Only 兼容性声明 + 警告
4. 若 target_agents == custom list:
     → 取列表的工具能力交集
     → 若交集含 Task → 用白名单 B + subagent
     → 否则 → 用白名单 A + 顺序执行
```
