# JSON 范例 & 断言速查

> 评估流程中所有结构化数据的**实战范例**（不写完整 JSON Schema 定义——没人会真用来校验）。
> 主控 SKILL.md 在生成/解析 JSON 时引用本文档照抄结构。

---

## 1. evals.json（测试用例集）

```json
{
  "skill_name": "gen-component",
  "skill_path": "skills/gen-component",
  "version": "0.1.0",
  "created_at": "2026-05-21T10:30:00Z",
  "evals": [
    {
      "id": "standard-1",
      "type": "standard",
      "name": "典型组件生成",
      "prompt": "/gen-component Button --type primary",
      "files": [],
      "expected_behavior": "在 src/components/Button/ 下生成 Button.tsx、Button.test.tsx、index.ts",
      "assertions": [
        { "type": "file_exists", "path": "src/components/Button/Button.tsx" },
        { "type": "file_contains", "path": "src/components/Button/Button.tsx", "pattern": "export const Button" },
        { "type": "tsc_passes", "files": ["src/components/Button/**/*.tsx"] },
        { "type": "eslint_passes", "files": ["src/components/Button/**/*.tsx"], "max_errors": 0 }
      ]
    }
  ]
}
```

**evals 数组最少 5 个**，类型分布建议 2 standard + 2 edge + 1 rejection + 1 stress。

---

## 2. 断言类型速查（12 种）

| 类型 | 必填字段 | 用途 | 客观/主观 |
|------|---------|------|----------|
| `file_exists` | `path` | 生成的文件存在 | 客观 |
| `file_not_exists` | `path` | 不该生成的文件没出现 | 客观 |
| `file_contains` | `path` + `pattern` (+ `is_regex`?) | 文件含指定内容 | 客观 |
| `file_not_contains` | `path` + `pattern` | 文件不含指定内容（如禁 `any`）| 客观 |
| `tsc_passes` | `files` (glob 数组) | TypeScript 编译零错误 | 客观 |
| `eslint_passes` | `files` + `max_errors` + `max_warnings`? | ESLint 错误数 ≤ 阈值 | 客观 |
| `test_passes` | `pattern` + `runner` (`vitest`/`jest`) | 指定测试通过 | 客观 |
| `output_contains` | `pattern` | 技能返回的文本含指定内容 | 客观 |
| `output_matches_schema` | `schema` (JSON Schema 对象) | 输出符合结构 | 客观 |
| `exit_code` | `value` (int) | 退出码匹配 | 客观 |
| `subjective_quality` | `criteria` (string) + `weight`? | 输出质量符合描述 | **主观**（grader）|
| `style_matches` | `reference_file` (path) | 输出风格与参考一致 | **主观**（grader）|

**优先级**：能用客观断言就不用主观断言。主观断言成本高、不稳定，仅在创意/分析类技能必要时使用。

---

## 3. grading.json（单个 eval 的评分结果）

```json
{
  "eval_id": "standard-1",
  "pass": true,
  "assertions": [
    { "type": "file_exists", "path": "src/components/Button/Button.tsx", "passed": true },
    { "type": "tsc_passes", "files": ["src/components/Button/**/*.tsx"], "passed": true, "evidence": "0 errors" },
    { "type": "eslint_passes", "files": ["..."], "passed": false, "evidence": "2 errors: no-unused-vars" }
  ],
  "duration_ms": 3421,
  "stdout_excerpt": "✅ 已生成 Button (3 files)",
  "stderr_excerpt": "",
  "bias_warning": false
}
```

Universal 模式跑主观断言时，`bias_warning: true`（提醒自评不可信）。

---

## 4. summary.json（整体评估汇总）

```json
{
  "skill_name": "gen-component",
  "iteration": 1,
  "execution_mode": "sequential",
  "evals_total": 8,
  "evals_passed": 6,
  "pass_rate": 0.75,
  "failures_by_type": {
    "tsc_passes": 1,
    "eslint_passes": 1
  },
  "duration_ms_total": 18234,
  "evals": [ /* 各 eval 的完整 grading.json */ ]
}
```

`execution_mode`：`sequential` (Universal) 或 `parallel-subagent` (Claude Code Only)。

---

## 5. trigger-evals.json（触发词测试集）

```json
{
  "skill_name": "gen-component",
  "description_version": "v1",
  "queries": [
    {
      "id": 1,
      "should_trigger": true,
      "query": "在 dashboard 上加个'导出 CSV'按钮，配 Tailwind 样式",
      "category": "standard"
    },
    {
      "id": 11,
      "should_trigger": false,
      "query": "我们的 Button 组件渲染太慢，怎么优化",
      "category": "near-miss-domain"
    }
  ]
}
```

`queries` 至少 20 条（10 should-trigger + 10 should-not-trigger）。`category` 枚举：`standard` / `casual` / `implicit` / `near-miss-keyword` / `near-miss-domain` / `unrelated`。

---

## 6. trigger-grading.json（触发词测试结果）

```json
{
  "description_version": "v1",
  "true_positive_rate": 0.8,
  "true_negative_rate": 0.9,
  "f1_score": 0.84,
  "results": [
    {
      "query_id": 1,
      "should_trigger": true,
      "actually_triggered": true,
      "reasoning": "命中 'dashboard 上加' + 描述新组件需求"
    }
  ],
  "failures_by_category": {
    "implicit": 2,
    "near-miss-keyword": 1
  }
}
```

---

## 7. 文件命名约定

| 文件 | 位置 |
|------|-----|
| `evals.json` | 技能根目录或 `evals/` 子目录 |
| `grading.json` | `${WORKSPACE}/<skill>/<eval-id>/grading.json` |
| `summary.json` | `${WORKSPACE}/<skill>/summary.json` |
| `trigger-evals.json` | `${WORKSPACE}/<skill>/trigger/queries.json` |
| `trigger-grading.json` | `${WORKSPACE}/<skill>/trigger/results.json` |
