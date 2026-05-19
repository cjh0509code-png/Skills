---
name: 图片生成工具
emoji: 🎨
category: AI 工具
skill: image-gen-skill
description: 在 Claude Code 中直接生成 AI 图片，支持火山引擎、阿里百炼、Gemini、Azure 四大供应商，自动优化提示词。
---

# image-gen — Claude Code 图片生成 Skill

在 Claude Code 中直接生成 AI 图片。支持火山引擎、阿里百炼、Google Gemini、Azure OpenAI 四大供应商，自动将任意描述/文档内容转为适合图片模型的详细英文提示词。

---

## 前置要求

- [Claude Code](https://claude.ai/code) 已安装
- Node.js 18+（`node --version` 确认）
- 至少一个供应商的 API Key

---

## 安装

**1. 安装 Skill**

```bash
npx skills add cjh0509code-png/Skills --skill image-gen-skill
```

**2. 配置 API Key**（至少一个，可以同时配置多个）

编辑 `~/.claude/settings.json`：

```json
{
  "env": {
    "ARK_API_KEY": "your-volcengine-key",
    "DASHSCOPE_API_KEY": "sk-your-dashscope-key",
    "GEMINI_API_KEY": "AIza-your-gemini-key",
    "AZURE_OPENAI_KEY": "your-azure-key",
    "AZURE_OPENAI_ENDPOINT": "https://your-resource.openai.azure.com"
  }
}
```

配置多个 Key 后，可用 `--provider` 随时切换供应商，无需改配置。

**3. 重启 Claude Code**

---

## 供应商与 API Key 获取

| 供应商 | 环境变量 | 获取地址 |
|--------|---------|---------|
| 火山引擎（volcengine_maas）| `ARK_API_KEY` | [console.volcengine.com/ark](https://console.volcengine.com/ark) |
| 阿里百炼（tongyi）| `DASHSCOPE_API_KEY` | [bailian.console.aliyun.com](https://bailian.console.aliyun.com) |
| Google Gemini（vertex_ai）| `GEMINI_API_KEY` | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) |
| Azure OpenAI（azure_openai）| `AZURE_OPENAI_KEY` + `AZURE_OPENAI_ENDPOINT` | [portal.azure.com](https://portal.azure.com) |

---

## 使用方法

### 基础生成

```
/image-gen 一只在草地上奔跑的柯基犬
```

Claude 会自动将中文描述转为详细的英文提示词，再调用图片模型生成。

### 从文件内容生成配图

```
/image-gen --from src/posts/react-performance.md
```

Claude 读取文件内容，分析主题和风格，自动生成适合该内容的配图提示词，再生成图片。

### 指定供应商

```
/image-gen a futuristic city --provider tongyi
```

### 指定保存位置

```
/image-gen 科技感十足的程序员 --save-dir public/images --filename programmer
```

### 查看配置状态

```
/image-gen --config
```

输出示例：
```
=== image-gen 配置状态 ===

[volcengine_maas] ✅ 已配置
  Key: ARK_API_KEY | Endpoint: https://ark.cn-beijing.volces.com

[tongyi] ❌ 未配置
  Key: DASHSCOPE_API_KEY | Endpoint: https://dashscope.aliyuncs.com
...
```

---

## 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `<描述>` | 必填 | 自然语言描述，支持中英文 |
| `--from <路径>` | — | 读取文件内容作为生成依据 |
| `--provider` | 自动（首个已配置）| volcengine_maas / tongyi / vertex_ai / azure_openai |
| `--model` | 供应商默认 | 见下表 |
| `--size` | 供应商默认 | volcengine_maas 最小 2048×2048 |
| `--save-dir` | 当前目录 | 相对或绝对路径，自动创建 |
| `--filename` | `image-{时间戳}` | 不含扩展名 |

## 各供应商默认值

| 供应商 | 默认模型 | 默认尺寸 | 最小尺寸 |
|--------|---------|---------|---------|
| volcengine_maas | Doubao-Seedream-3.0 | 2048×2048 | 2048×2048 |
| tongyi | wanx2.1-t2i-turbo | 1024×1024 | 512×512 |
| vertex_ai | gemini-2.0-flash-preview-image-generation | 模型自适应 | — |
| azure_openai | gpt-image-1 | 1024×1024 | 256×256 |

---

## 提示词自动优化

无论用户输入什么（简单描述、文档内容、中文），skill 都会先将其转为结构化的详细英文提示词：

```
[风格/质量] [主体] [动作/状态] [场景/背景] [光线/色彩] [构图] [技术参数]
```

示例：
- 输入：`一只猫`
- 优化后：`a fluffy orange tabby cat sitting gracefully on a wooden windowsill, warm afternoon sunlight, shallow depth of field, photorealistic, 8k detail, no text, no watermark`

---

## 常见问题

**Q：多个供应商 Key 都配置了，哪个会被使用？**

未指定 `--provider` 时，按 volcengine_maas → tongyi → vertex_ai → azure_openai 顺序选第一个已配置的。

**Q：火山引擎生成报错"尺寸太小"？**

火山引擎最小 2048×2048，skill 会自动调整，无需手动设置。

**Q：图片保存在哪里？**

默认保存到 Claude Code 的当前工作目录（你打开的项目根目录）。用 `--save-dir` 指定其他位置。

**Q：`--from` 支持什么格式的文件？**

任意文本文件（.md、.txt、.html 等），Claude 读取后分析内容生成配图。

---

## 文件说明

```
image-gen-skill/
├── SKILL.md    # Claude Code skill，复制到 ~/.claude/skills/image-gen/ 使用
└── README.md   # 本文档
```
