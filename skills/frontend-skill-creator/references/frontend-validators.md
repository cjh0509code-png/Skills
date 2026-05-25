# 前端验证器集成（跨平台）

> 主控 SKILL.md 阶段 7 引用本文档调用 TypeScript / ESLint / Vitest / Playwright / Stylelint 等。
>
> **跨平台原则**：所有验证器统一用 §2 通用驱动器调用，避免 bash heredoc / `tee` / `$VAR` 等 PowerShell 不兼容写法。

---

## 1. 验证器索引

| 验证器 | 触发的断言 | 必要条件 | 底层 npx 命令 |
|--------|----------|---------|--------------|
| TypeScript | `tsc_passes` | 项目有 `tsconfig.json` | `npx tsc --noEmit` |
| ESLint | `eslint_passes` | 项目有 ESLint 配置 | `npx eslint --format json` |
| Vitest | `test_passes` (vitest) | 项目用 Vitest | `npx vitest run --reporter=json --outputFile=…` |
| Jest | `test_passes` (jest) | 项目用 Jest | `npx jest --json --outputFile=…` |
| Playwright | `e2e_passes` | E2E 项目 | `npx playwright test --reporter=json` |
| Stylelint | `stylelint_passes` | CSS 项目 | `npx stylelint --formatter json` |
| size-limit | `bundle_size_ok` | 配过 size-limit | `npx size-limit --json` |

---

## 2. 通用驱动器（所有验证器都用这个调）

跨平台 Node.js 包装，把 `npx <cmd>` 输出标准化为 `{ passed, evidence, raw }`：

```bash
node --input-type=module -e "
import { spawnSync } from 'node:child_process';

const cwd = process.env.PROJECT_ROOT || process.cwd();
const cmd = process.env.CMD;            // e.g. 'tsc'
const args = JSON.parse(process.env.ARGS || '[]');
const timeoutMs = parseInt(process.env.TIMEOUT_MS || '60000', 10);

const r = spawnSync('npx', [cmd, ...args], {
  cwd, encoding: 'utf8', shell: true, timeout: timeoutMs,
});

process.stdout.write(JSON.stringify({
  passed: r.status === 0,
  exit: r.status,
  stdout: (r.stdout || '').slice(-3000),
  stderr: (r.stderr || '').slice(-1000),
  timed_out: r.signal === 'SIGTERM',
}));
"
```

**关键点**：
- `spawnSync` 跨平台；`shell: true` 让 Windows 找到 `.cmd`/`.bat` 形态的 npx
- timeout 默认 60s，可按需覆盖
- 不写中间临时文件到 `/tmp/...`，结果直接 stdout

---

## 3. 各验证器差异（仅列**与通用驱动器不同**的部分）

### TypeScript（`tsc_passes`）

- **CMD**：`tsc`
- **ARGS**：`["--noEmit", "--pretty", "false"]`
- **TIMEOUT_MS**：120000
- **特殊解析**：从 stdout 用 regex `/error TS\d+/` 抓错误行：

```js
const errors = stdout.split(/\r?\n/).filter(l => /error TS\d+/.test(l));
// passed = errors.length === 0
```

- **限定范围检查**（仅查生成文件）：写临时 tsconfig 到 `os.tmpdir()`：

```js
import fs from 'node:fs'; import path from 'node:path'; import os from 'node:os';
const tmpCfg = path.join(os.tmpdir(), 'fsc-tsc-' + Date.now() + '.json');
fs.writeFileSync(tmpCfg, JSON.stringify({
  extends: path.join(cwd, 'tsconfig.json'),
  include,
  exclude: ['**/*.test.*'],
  compilerOptions: { baseUrl: cwd },  // ⚠️ 必须显式回指 cwd，否则 alias 解析挂
}));
// 然后 ARGS = ['--noEmit', '-p', tmpCfg]; 用完 unlinkSync(tmpCfg)
```

### ESLint（`eslint_passes`）

- **CMD**：`eslint`
- **ARGS**：`["--format", "json", ...files]`
- **特殊解析**：ESLint 即使有错也输出 JSON 到 stdout，需 try/catch 解析：

```js
let parsed;
try { parsed = JSON.parse(r.stdout); }
catch { return { passed: false, error: 'eslint_no_config_or_crash', stderr }; }
const errors = parsed.reduce((s, f) => s + (f.errorCount || 0), 0);
const warnings = parsed.reduce((s, f) => s + (f.warningCount || 0), 0);
// passed = errors === 0 (或 ≤ max_errors)
```

- **配置缺失 fallback**：沙盒目录常没 ESLint config。先把项目根的配置拷贝到沙盒：

```js
const candidates = ['.eslintrc.cjs','.eslintrc.js','.eslintrc.json','eslint.config.js','eslint.config.mjs'];
for (const f of candidates) {
  const src = path.join(projectRoot, f);
  if (fs.existsSync(src)) fs.copyFileSync(src, path.join(sandbox, f));
}
```

- **自动修复**：CMD 改 `eslint`，ARGS 加 `--fix`，stdio: 'inherit'

### Vitest（`test_passes` runner=vitest）

- **CMD**：`vitest`
- **ARGS**：`["run", "--reporter=json", "--outputFile=" + outFile, ...(pattern ? [pattern] : [])]`
- **TIMEOUT_MS**：180000
- **关键**：用 `--outputFile=os.tmpdir()/fsc-vitest-XXX.json` **不要**靠 stdout 重定向（vitest 的 verbose 日志会污染 JSON）
- **特殊解析**：从 outFile 读 JSON，提取 `numPassedTests` / `numFailedTests` / `testResults[].testResults[].status`

### Jest（`test_passes` runner=jest）

- **CMD**：`jest`
- **ARGS**：`["--json", "--outputFile=" + outFile, ...]`
- 其余同 Vitest

### Playwright（`e2e_passes`）

- **CMD**：`playwright`
- **ARGS**：`["test", "--reporter=json"]`
- **TIMEOUT_MS**：600000（E2E 慢）

### Stylelint（`stylelint_passes`）

- **CMD**：`stylelint`
- **ARGS**：`["--formatter", "json", ...files]`
- 解析方式类似 ESLint

---

## 4. 跨技术栈适配规则

按 Q1 收集的技术栈决定该跑哪些验证器：

```javascript
const validatorMap = {
  language: { typescript: ['tsc_passes'], javascript: [] },
  linter:   { eslint: ['eslint_passes'], biome: ['biome_passes'], none: [] },
  test_runner: { vitest: ['test_passes'], jest: ['test_passes'], none: [] },
};
```

技能生成时（阶段 6），自动把对应断言写入 evals.json。

---

## 5. Fallback 策略

| 情况 | 处理 |
|------|------|
| 无 `package.json` | 跳过所有验证，标"非项目目录" |
| 无 `tsconfig.json` | 跳过 `tsc_passes`，标"无 tsconfig" |
| 无 ESLint 配置（项目根也没）| 跳过 `eslint_passes`，标"无 ESLint 配置" |
| ESLint 配置存在但沙盒缺 | 按 §3 ESLint fallback 拷贝后重试 |
| `npx` 命令未找到 | 报错 + 提示安装 Node.js 18+ |
| 验证器超时 | 标 timeout 失败，下一项 |
| 验证器输出非 JSON 崩溃 | 标 "validator_crashed"，附 stderr 摘要 |
| Windows `spawn ENOENT` | 检查 `shell: true`（必须）|

---

## 6. 平台兼容性矩阵

| 平台 | `node --input-type=module -e "<单行>"` | `<多行>` | `spawnSync('npx',...,{shell:true})` |
|------|------|------|------|
| Linux / macOS / Git Bash | ✅ | ✅ | ✅ |
| Windows + PowerShell | ✅ | ⚠️ 多行需 `` ` `` 续行 + 内嵌单引号转义注意 | ✅ |
| Windows + cmd（原生）| ✅ | ⚠️ 多行需 `^` 续行；推荐先 Write 工具写 `${WORKSPACE}/script.mjs` 再 `node script.mjs` | ✅ |

**结论与最低要求**：
- **最低环境**：Git Bash / PowerShell / macOS / Linux
- **原生 cmd 用户最稳方案**：先 Write 工具落盘 `script.mjs`，再 `node script.mjs`（避免多行字符串转义）

---

## 7. 输出给用户的简化版

不暴露 JSON 中间产物，对话内只展示：

```
🧪 验证结果
  TypeScript：✅ 0 errors
  ESLint：⚠️ 1 warning (no-unused-vars: src/components/Button/Button.tsx:3)
  Vitest：✅ 3/3 tests passed
```
