# 触发词优化（Description Tuning）

> 阶段 10 引用本文档。
> 目的：让生成的技能 description 既"足够推销"（避免欠触发），又"足够精准"（避免误触发）。

---

## 为什么重要

技能是否被 Claude 主动调用，**90% 取决于 description**：
- description 太保守 → Claude 不知道何时该用 → **欠触发（undertrigger）**
- description 太宽泛 → Claude 在不相关场景滥用 → **误触发（overtrigger）**

Anthropic 官方 skill-creator 指出：当下 Claude 倾向于欠触发，所以 description 要**有点"推销感"**。

---

## 测试集结构

为每个技能生成 **20 个测试 query**：
- **10 个 should-trigger**（应触发）：不同措辞、不同上下文、含隐式需求
- **10 个 should-not-trigger**（不应触发）：near-miss，共享关键词但意图不同

```json
{
  "skill_name": "gen-component",
  "description_version": "v1",
  "queries": [
    {
      "id": 1,
      "should_trigger": true,
      "query": "帮我在 src/components 下加一个 Button 组件，要 primary 和 secondary 两种样式，配上 Storybook"
    },
    {
      "id": 11,
      "should_trigger": false,
      "query": "Button 组件的样式应该用什么颜色才符合 Material Design 规范？"
    }
  ]
}
```

---

## 生成 should-trigger 用例的原则

8-10 个用例，覆盖：

1. **不同措辞**：同一意图用 3-5 种说法
   - "做一个 Button 组件"
   - "我需要一个 Button"
   - "帮我建个 Button"
   - "在 components 下新建 Button"

2. **含上下文细节**：贴近真实使用场景
   - ❌ Bad: "make a button"（太抽象）
   - ✅ Good: "客户要 dashboard 上加个'导出 CSV'按钮，放在右上角，hover 时加阴影，用我们项目的 Tailwind classes"

3. **隐式需求**：不直接说"创建技能名"也要触发
   - "src/components 下缺一个数据表格组件，能不能帮我搭个骨架"
   - "下个 sprint 要做几十个 form field 组件，能不能批量来一波"

4. **边界用例**：易被误判为不触发的，但确实该触发
   - "我想抽出一个新组件"（不明说 generate，但意图明确）

---

## 生成 should-not-trigger 用例的原则

8-10 个 near-miss，覆盖：

1. **共享关键词但意图不同**：
   - "Button 组件的样式应该用什么颜色"（讨论设计，不是生成）
   - "我们现在的 Button 组件有性能问题，怎么优化"（debug，不是生成）

2. **相邻领域**：
   - "怎么用 Storybook 写 Button 的 doc"（讨论 Storybook 用法，不是生成 story 文件）

3. **超出范围**：
   - "把 Button.vue 改成 React 组件"（这是 Transformer 的活，不是 Generator）

4. **明显不相关但含部分关键词**：
   - "今天去公司路上看到 Button 按错被电梯门夹了下"（玩笑，含 Button 关键词但无关）

---

## 评估流程（手动版）

1. 生成测试集 JSON → 展示给用户
2. 用户审核 → 增删改 query
3. 对每个 query：
   - 模拟"假设 Claude 看到这条 query + 当前 description，会调用技能吗？"
   - 由用户判定（或由你扮演 Claude 给出 yes/no + 理由）
4. 计算：
   - **真阳性率**（should_trigger 中实际触发的比例）
   - **真阴性率**（should_not_trigger 中实际不触发的比例）
5. 失败的 query 聚类 → 给出 description 优化建议

---

## 优化建议的形式

不自动改 description，给用户**3 个改进版本**让用户选：

```
当前 description：
> [当前版本]

测试结果：should-trigger 7/10，should-not-trigger 8/10

失败分析：
  • should-trigger 失败：3 条都是"隐式生成需求"，description 没强调"主动用"
  • should-not-trigger 失败：2 条都是"讨论组件"被误触发

3 个优化版本：

A. 加强主动触发（修复欠触发）：
> [版本 A，加入"哪怕用户没明说 generate / create，只要意图是新建组件就用"]

B. 加强排除（修复误触发）：
> [版本 B，明确"只在用户要新建文件时用，不用于讨论或 debug"]

C. 综合优化（推荐）：
> [版本 C，两者兼顾]

你选哪个？或者要我再生成几个？
```

---

## "推销式" description 的写法套路

1. **明确做什么**：一句话说清楚（动词 + 对象）
2. **列具体场景**：3-5 个具体使用时机
3. **加 pushy 句**：
   - "务必在以下场景触发此技能，即使用户没有明确说[关键词]"
   - "只要用户提到[场景词]，就应该用此技能"
4. **列排除场景**（避免误触发）：
   - "不用于：讨论设计、debug、迁移现有代码"

### 示例 1：Generator + Component（React Button）

```yaml
description: |
  在 src/components/ 下生成新 React 组件，含 .tsx + .test.tsx + .stories.tsx + index.ts。

  务必在以下场景触发，即使用户没说"generate"或"create"：
  • 用户说"加一个 X 组件"、"做个 X"、"我需要一个 X"
  • 用户说"在 components 下新建"、"抽出一个新组件"
  • 用户描述了一个新组件的需求和样式细节

  不用于：
  • 讨论现有组件的设计、性能、样式调整
  • 把组件从其他框架迁移过来（用 vue-to-react-migrator 之类）
  • 修改/重构已存在的组件
  • 生成 Hook / Composable（用 gen-hook / gen-composable）
```

### 示例 2：Analyzer（a11y 审查）

```yaml
description: |
  扫描 React/Vue 项目的无障碍性（a11y）问题，输出 markdown 审查报告 + 自动修复建议。
  检查规则覆盖 WCAG 2.1 AA 的常见 issue：缺 alt、缺 label、缺 aria 角色、对比度不足、
  键盘不可达、焦点管理缺失等。

  务必在以下场景触发，即使用户没明确说"a11y"或"accessibility"：
  • 用户说"看下这个页面有没有无障碍问题"、"做个无障碍审查"
  • 用户问"为什么 screen reader 读不出来"、"按 tab 跳不到这个按钮"
  • 用户说"准备发布前过一遍 a11y"、"过一下 WCAG 检查"
  • 用户描述了视障/键盘用户的 case 想验证

  不用于：
  • 生成新组件（即使要求"a11y-ready 的组件" —— 用 gen-component 配合 a11y 模板）
  • 修复已知的具体 a11y bug（直接编辑代码即可）
  • 讨论 a11y 设计原则
```

> 其他类型示例（Composable / Transformer）见 git 历史。模式不变，只是触发场景换成对应技能类型。

---

## 集成到主控

主控 SKILL.md 的阶段 10 输出：

```
🎯 触发词优化 (Description Tuning)

我已经生成了 20 个测试 query（10 个应触发 + 10 个不应触发）：

[Markdown 表格展示用例]

我会扮演 Claude，对每条 query 判断"会不会用这个 skill"，然后给出 description 改进建议。

要继续吗？也可以先增删改这些 query。
```
