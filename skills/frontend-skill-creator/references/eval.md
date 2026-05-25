# 评估协议（合并 eval-protocol + eval-execution-modes）

> 主控 SKILL.md 阶段 6（生成测试用例）、7（沙盒执行）、8（评分聚类）、9（迭代）引用本文档。
> JSON 范例与断言速查见 `schemas.md`。

---

## 1. 测试用例集设计

每个生成的技能至少配 **5-8 个 evals**，覆盖 4 类场景：

| 类型 | 数量 | 目的 |
|------|-----|------|
| `standard` | 2 | 典型真实输入，验证主流程 |
| `edge` | 2 | 模糊/不完整输入，验证 Fallback |
| `rejection` | 1 | 超范围输入，验证礼貌拒绝 |
| `stress` | 1 | 超长/复杂输入，验证稳定性 |
| `regression` | 可选 | 历史 bug 复现，防回滚 |

evals.json 结构和断言类型见 `schemas.md §1-2`。**优先选客观断言**（`tsc_passes` / `eslint_passes` / `file_exists` 等）。

---

## 2. 工作目录布局

所有产物在 `${WORKSPACE}` 下（WORKSPACE 定义见主 SKILL.md "工作目录决议"）：

```
${WORKSPACE}/<skill>/
├── SKILL.md.draft       ← 草稿（阶段 5 输出）
├── evals.json           ← 测试用例集（阶段 6 输出）
├── eval-<id>/
│   ├── inputs/          ← fixture（拷贝自 eval.files）
│   ├── outputs/         ← 技能生成的文件
│   ├── stdout.log
│   ├── stderr.log
│   └── grading.json     ← 评分（阶段 7 输出，结构见 schemas.md §3）
└── summary.json         ← 整体汇总（阶段 8 输出，结构见 schemas.md §4）
```

---

## 3. 阶段 7 协议 A：顺序执行（Universal / 无 Task 场景）

**适用**：默认模式 / `custom-list` 中含无 `Task` 能力的 agent。

**优势**：跨平台。**代价**：主任务自评有偏见、不能并行。

### 流程

```
for eval in evals:
  1. 创建隔离工作目录：${WORKSPACE}/<skill>/eval-<id>/
  2. 拷贝 eval.files 中的 fixture 到 inputs/
  3. **上下文重置指令**（关键）："以下执行 eval-<id>，仅基于即将给出的
     SKILL.md draft 和 eval.prompt 回答，不参考此前对话产生的任何状态。"
  4. Claude 自己化身为草稿技能，按 SKILL.md.draft 工作流执行 eval.prompt
  5. 把生成产物（文件、stdout）保存到 outputs/
  6. 对每个 assertion 跑判定器：
     - 客观断言 → Read frontend-validators.md 跑 CLI 验证器
     - 主观断言 → Claude 自评 + 在 grading.json 加 `bias_warning: true`
  7. 写 grading.json（结构见 schemas.md §3）
  8. **状态清理**：移除临时变量引用，准备下一个 eval
```

### 偏见缓解

主观断言自评有偏见风险（Claude 既是执行者又是评分者）。Universal 模式只能：
- 在 grading.json 加 `bias_warning: true`
- 在 summary.json 标 `execution_mode: "sequential"`
- 建议用户人工复核主观断言结果

---

## 4. 阶段 7 协议 B：subagent 隔离并行（Claude Code Only）

**适用**：`target_agents = claude-code-only` 或 custom-list 含 Task 能力的 agent。

**优势**：并行 + 隔离 + 独立 grader（无自评偏见）。**代价**：平台锁定 Claude Code、N × token 消耗。

### 流程

```
对每个 eval（可并行 spawn）：
  1-2. 同协议 A
  3. spawn Claude Code Task subagent（模板见下）
  4. 等待 subagent 返回，收集 JSON + Task notification 中的 token / duration
  5. 对每个 assertion 跑判定器：
     - 客观断言 → 主任务跑（同协议 A）
     - 主观断言 → **另起 grader subagent**（模板见下）
  6. 写 grading.json，`bias_warning: false`（grader 独立无偏见）
```

### 子代理模板（执行 eval）

```
工具: Task
subagent_type: general-purpose
description: "Execute eval-<id> for <skill-name>"
prompt: |
  你是 <skill-name> 这个技能的执行者。
  1. Read 草稿 SKILL.md：${WORKSPACE}/<skill-name>/SKILL.md.draft
  2. 切换到工作目录：${WORKSPACE}/<skill-name>/eval-<id>/
  3. 严格按 SKILL.md 的工作流，对以下用户输入执行任务：
     ---
     <eval.prompt 内容>
     ---
  4. 把生成的所有文件保存到 outputs/ 子目录
  5. 最后返回 JSON: { files_created, stdout_excerpt, errors }

  禁止：超出 SKILL.md 工作流；脑补能力；修改 SKILL.md 外的文件；联网。
  超时：5 分钟。
```

### Grader 子代理模板（评主观断言）

```
工具: Task
subagent_type: general-purpose
prompt: |
  对照标准 '<criteria>' 评估以下输出：
  ---
  <subagent 输出内容>
  ---
  返回 JSON: { passed: true|false, evidence: '<理由 50 字内>' }
```

---

## 5. 错误恢复（两种协议通用）

| 情况 | 处理 |
|------|------|
| 某 eval 失败（生成不出文件 / 验证器崩） | 记录 grading.json `failure_reason`，继续下一个 |
| 验证器（npx tsc 等）缺失 | 跳过该断言，标注"环境缺工具"，**不算失败** |
| Universal 模式上下文严重污染 | 提醒用户：建议改用 Claude Code 的 subagent 模式 |
| Subagent 超时（>5min）| 标记 timeout，继续 |
| Subagent 拒绝执行 | 标记 refused，记录拒绝理由 |
| Grader subagent 返回非法 JSON | 重试 1 次，仍失败则标记 grader_error |
| 工作目录创建失败 | fallback 到 `${TMPDIR}/fsc-eval/<skill>/` |
| Claude Code Only 选项 + 当前 agent 不是 Claude Code | 警告 + 询问是否降级为协议 A |

---

## 6. 决策速查

| `target_agents` | 走哪个协议 |
|----------------|----------|
| `universal`（默认）| §3 顺序执行 |
| `claude-code-only` | §4 subagent 隔离 |
| `custom-list` 全含 Task | §4 |
| `custom-list` 任一无 Task | §3（取最小公共能力集）|

---

## 7. 阶段 8：评分聚类

1. 汇总所有 `grading.json` 到 `summary.json`
2. 计算 `pass_rate`
3. 失败用例按以下维度聚类：
   - **按断言类型**：哪类断言最常失败？（如 `tsc_passes` 多 → 模板生成有类型问题）
   - **按规则**：哪条审查规则误报多？
   - **按文件**：是否集中在某文件类型？

展示给用户：
```
📊 评估完成（第 N 轮，模式：sequential|parallel-subagent）

通过率：X/Y (Z%)

失败聚集：
  • [断言类型]：[次数] 次失败 — [典型原因]

[每条建议改什么]
```

---

## 8. 阶段 9：迭代决策

| 轮次 | pass_rate | 决策 |
|------|-----------|------|
| 第 1 轮 | ≥ 0.9 | 询问满意 → 满意直接进阶段 10；不满意继续 |
| 第 1 轮 | 0.6-0.9 | 自动改 SKILL.md → 回阶段 7 |
| 第 1 轮 | < 0.6 | 提示设计可能有问题，建议回 Q1-Q4 |
| 第 2-3 轮 | ≥ 0.9 | 直接进阶段 10 |
| 第 2-3 轮 | 0.6-0.9 | 必须有 ≥ 5% 改善才继续；否则进阶段 10 + 报告剩余问题 |
| 第 2-3 轮 | < 0.6 | 进阶段 10 + 在最终报告里明确"基础设计需要重做" |

**最多 3 轮迭代**。第 2 轮起每轮 pass_rate 必须有 ≥ 5% 改善，否则停止并交付当前版本。

每轮改进聚焦：模板措辞 / few-shot 示例 / 验证范围调整 / 代码模板修正。
