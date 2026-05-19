---
name: image-gen
description: |
  AI 图片生成工具。支持火山引擎(volcengine_maas)、阿里百炼(tongyi)、
  Google Gemini(vertex_ai)、Azure OpenAI(azure_openai) 四大供应商。
  支持多供应商 Key 同时配置、默认值自动填充、将任意内容/文档/想法
  智能转为适合图片模型的详细英文提示词再生成。
  零硬编码，凭证通过环境变量注入。
  触发条件：/image-gen，或"生成图片"、"画一张"、"generate image"、"帮我生成配图"
triggers:
  - image-gen
allowed-tools: Bash, Read, Glob
---

# image-gen — AI 图片生成工具

## 命令速查

| 用法 | 说明 |
|------|------|
| `/image-gen <描述>` | 直接生成（自动优化提示词） |
| `/image-gen --from <文件路径>` | 读取文件内容，生成配图 |
| `/image-gen <描述> --provider tongyi` | 指定供应商 |
| `/image-gen <描述> --model wanx2.1-t2i-turbo` | 指定模型 |
| `/image-gen <描述> --size 1024x1024` | 指定尺寸 |
| `/image-gen <描述> --save-dir public/images/` | 指定保存目录 |
| `/image-gen <描述> --filename my-pic` | 指定文件名（不含扩展名）|
| `/image-gen --config` | 查看所有供应商配置状态 |
| `/image-gen --help` | 显示帮助 |

---

## 默认值

| 参数 | 默认值 |
|------|--------|
| `--provider` | 自动选择第一个已配置 Key 的供应商（优先 volcengine_maas）|
| `--model` | 各供应商独立默认（见下表）|
| `--size` | 各供应商独立默认（见下表）|
| `--save-dir` | 当前工作目录（`$PWD`）|
| `--filename` | `image-{unix时间戳}` |

### 各供应商默认值

| Provider | 默认模型 | 默认尺寸 | 尺寸规则 | 所需 Key |
|----------|---------|---------|---------|---------|
| `volcengine_maas` | `Doubao-Seedream-3.0` | `2048x2048` | **宽×高 ≥ 3,686,400 像素**。示例：`1920x1920` `2048x2048` `2560x1440` `1440x2560` | `ARK_API_KEY` |
| `tongyi` | `wanx2.1-t2i-turbo` | `1024*1024` | 格式用 `*` 分隔（传 `x` 自动转换）。示例：`512*512` `1024*1024` `768*1152` `1152*768` | `DASHSCOPE_API_KEY` |
| `vertex_ai` | `gemini-2.0-flash-preview-image-generation` | `1:1` | 按**宽高比**指定：`1:1` `3:4` `4:3` `9:16` `16:9` | `GEMINI_API_KEY` |
| `azure_openai` | `gpt-image-1` | `1024x1024` | 支持：`1024x1024` `1024x1536` `1536x1024` | `AZURE_OPENAI_KEY` + `AZURE_OPENAI_ENDPOINT` |

**供应商自动回退顺序**：volcengine_maas → tongyi → vertex_ai → azure_openai（取第一个已配置 Key 的）

---

## 多供应商 Key 同时配置

所有 Key 可以同时设置，随时用 `--provider` 切换：

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

---

## 提示词智能生成（核心能力）

**所有生成请求都经过提示词优化**，包括：

1. 用户直接给 prompt（如"一只猫"）
2. 用户给文件路径（`--from doc.md`）
3. 用户给一段文字内容（粘贴的文档、想法、需求）

### 优化流程

**第一步：理解意图**

分析用户输入，理解：
- 图片的用途（封面图、插图、背景、图标…）
- 主体内容（人物、物体、场景、概念…）
- 风格偏好（写实、插画、3D、水墨…）
- 情感基调（活泼、严肃、温暖、科技感…）

若输入是文档/文章内容，提取：文章主题、核心观点、目标读者、情感基调。

**第二步：生成详细英文提示词**

图像模型对英文理解更准确，生成时遵循以下结构：

```
[风格/质量] [主体描述] [动作/状态] [场景/背景] [光线/色彩] [构图/视角] [技术参数]
```

示例转换：
- 用户输入：`一只猫`
- 优化后：`a fluffy orange tabby cat sitting gracefully on a wooden windowsill, warm afternoon golden hour sunlight streaming through curtains, shallow depth of field, photorealistic, 8k detail, no text, no watermark`

- 用户输入：（某篇关于 React 性能优化的文章）
- 优化后：`abstract digital visualization of data flow optimization, glowing network nodes connected by light streams, dark blue and cyan color scheme, modern tech aesthetic, clean geometric shapes, 3D render, professional illustration, no text`

**第三步：展示并确认**

向用户展示：
```
📝 原始输入：[用户输入摘要]
✨ 优化提示词：[英文提示词]
🎨 供应商：[provider] / 模型：[model] / 尺寸：[size]
💾 保存位置：[路径]

正在生成...
```

然后直接生成（不需要用户二次确认，除非用户明确要求先预览提示词）。

---

## 执行步骤

### 步骤 1：解析参数

- `--from <路径>`：用 Read 工具读取文件，内容作为生成依据
- `--provider`：指定供应商；未指定则自动选择已配置的
- `--model`：指定模型；未指定则用供应商默认
- `--size`：volcengine_maas 填 `宽x高`（像素乘积 < 3,686,400 自动调整）；tongyi 填 `宽*高` 或 `宽x高`（自动转换）；vertex_ai 填宽高比如 `16:9`
- `--save-dir`：保存目录；未指定则用当前目录
- `--filename`：文件名（不含扩展名）；未指定则用时间戳
- 剩余内容（非 `--` 开头）：用户的原始描述/提示词

### 步骤 2：检查环境变量

```bash
TARGET_PROVIDER="<provider>" node -e "
const p = process.env.TARGET_PROVIDER;
const m = { volcengine_maas:'ARK_API_KEY', tongyi:'DASHSCOPE_API_KEY', vertex_ai:'GEMINI_API_KEY', azure_openai:'AZURE_OPENAI_KEY' };
const k = m[p];
if (!k) { console.error('未知供应商: '+p); process.exit(1); }
if (!process.env[k]) {
  console.error('缺少环境变量: '+k+'\n请在 ~/.claude/settings.json 的 env 字段添加:\n'+JSON.stringify({env:{[k]:'your-key'}},null,2));
  process.exit(1);
}
if (p==='azure_openai' && !process.env.AZURE_OPENAI_ENDPOINT) { console.error('缺少 AZURE_OPENAI_ENDPOINT'); process.exit(1); }
console.log('ok');
"
```

### 步骤 3：调用 API

```bash
PROMPT_TEXT="<优化后的英文提示词>" PROVIDER="<provider>" MODEL="<model>" SIZE="<size>" \
node --input-type=module << 'EOF'
const PROMPT = process.env.PROMPT_TEXT;
const PROVIDER = process.env.PROVIDER || 'volcengine_maas';
const SIZE = process.env.SIZE;
const MODEL = process.env.MODEL;

const cfg = {
  volcengine_maas: {
    endpoint: 'https://ark.cn-beijing.volces.com/api/v3/images/generations',
    headers: () => ({ Authorization: `Bearer ${process.env.ARK_API_KEY}` }),
    defaultModel: 'Doubao-Seedream-3.0', defaultSize: '2048x2048',
    buildBody(p, m, s) {
      // 像素乘积 < 3,686,400 自动调整为 2048x2048
      const [w, h] = (s || '2048x2048').split('x').map(Number);
      const sz = (w * h >= 3686400) ? s : '2048x2048';
      if (sz !== s) process.stderr.write(`⚠️  尺寸 ${s} 像素不足，已自动调整为 2048x2048\n`);
      return { model: m, prompt: p, size: sz, response_format: 'url', n: 1 };
    },
    extractUrl: d => d.data?.[0]?.url, ext: 'jpg',
  },
  tongyi: {
    endpoint: 'https://dashscope.aliyuncs.com/compatible-mode/v1/images/generations',
    headers: () => ({ Authorization: `Bearer ${process.env.DASHSCOPE_API_KEY}` }),
    defaultModel: 'wanx2.1-t2i-turbo', defaultSize: '1024x1024',
    buildBody: (p, m, s) => ({ model: m, prompt: p, size: (s||'1024x1024').replace('*','x'), n: 1 }),
    // 兼容标准格式和 DashScope 自定义格式
    extractUrl: d => d.data?.[0]?.url || d.output?.choices?.[0]?.message?.content?.[0]?.image,
    ext: 'jpg',
  },
  vertex_ai: {
    endpoint: () => `https://generativelanguage.googleapis.com/v1beta/models/${MODEL || 'gemini-2.0-flash-preview-image-generation'}:generateContent?key=${process.env.GEMINI_API_KEY}`,
    headers: () => ({}), defaultModel: 'gemini-2.0-flash-preview-image-generation', defaultSize: '1:1',
    buildBody: (p, m, s) => ({
      contents:[{role:'user',parts:[{text:p}]}],
      generationConfig:{ responseModalities:['TEXT','IMAGE'], candidateCount:1, imageConfig:{ aspectRatio: s||'1:1' } }
    }),
    extractBase64: d => { for(const part of d.candidates?.[0]?.content?.parts||[]) if(part.inlineData?.data) return part.inlineData.data; },
    ext: 'png',
  },
  azure_openai: {
    endpoint: () => `${process.env.AZURE_OPENAI_ENDPOINT}/openai/images/generations?api-version=2024-05-01-preview`,
    headers: () => ({ 'api-key': process.env.AZURE_OPENAI_KEY }),
    defaultModel: 'gpt-image-1', defaultSize: '1024x1024',
    buildBody: (p, m, s) => ({ model: m, prompt: p, size: s||'1024x1024', n: 1 }),
    extractUrl: d => d.data?.[0]?.url, extractBase64: d => d.data?.[0]?.b64_json, ext: 'png',
  },
};

const p = cfg[PROVIDER];
if (!p) { console.error('未知供应商:', PROVIDER); process.exit(1); }
const resolvedModel = MODEL || p.defaultModel;
const resolvedSize = SIZE || p.defaultSize;
const endpoint = typeof p.endpoint === 'function' ? p.endpoint() : p.endpoint;
const body = p.buildBody(PROMPT, resolvedModel, resolvedSize);
const res = await fetch(endpoint, { method:'POST', headers:{'Content-Type':'application/json',...p.headers()}, body:JSON.stringify(body) });
const data = await res.json();
if (!res.ok) { console.error('API 错误:', JSON.stringify(data,null,2)); process.exit(1); }
const url = p.extractUrl?.(data);
const base64 = p.extractBase64?.(data);
if (!url && !base64) { console.error('无法提取图片:', JSON.stringify(data,null,2)); process.exit(1); }
process.stdout.write(JSON.stringify({ provider:PROVIDER, model:resolvedModel, url, base64, ext:p.ext }));
EOF
```

### 步骤 4：保存图片

```bash
SAVE_DIR="<绝对路径>" FILENAME="<文件名>" \
node --input-type=module << 'EOF'
import fs from 'fs'; import path from 'path';
const raw = await new Promise(r => { let d=''; process.stdin.on('data',c=>d+=c); process.stdin.on('end',()=>r(d)); });
const result = JSON.parse(raw);
const dir = process.env.SAVE_DIR || process.cwd();
const name = process.env.FILENAME || `image-${Date.now()}`;
const file = path.join(dir, `${name}.${result.ext}`);
fs.mkdirSync(dir, { recursive: true });
if (result.url) { const r=await fetch(result.url); fs.writeFileSync(file, Buffer.from(await r.arrayBuffer())); }
else if (result.base64) { fs.writeFileSync(file, Buffer.from(result.base64,'base64')); }
const size = fs.statSync(file).size;
console.log(`✅ 图片已保存\n路径: ${path.resolve(file)}\n大小: ${(size/1024).toFixed(1)} KB\n供应商: ${result.provider} / 模型: ${result.model}`);
EOF
```

管道连接（步骤 3 → 步骤 4）：

```bash
PROMPT_TEXT="..." PROVIDER="..." MODEL="..." SIZE="..." node --input-type=module << 'GENEOF' ... GENEOF \
| SAVE_DIR="..." FILENAME="..." node --input-type=module << 'SAVEEOF' ... SAVEEOF
```

---

## --config 模式

```bash
node --input-type=module << 'EOF'
const providers = [
  {name:'volcengine_maas', key:'ARK_API_KEY',      endpoint:'https://ark.cn-beijing.volces.com',             extra:[]},
  {name:'tongyi',          key:'DASHSCOPE_API_KEY', endpoint:'https://dashscope.aliyuncs.com',               extra:[]},
  {name:'vertex_ai',       key:'GEMINI_API_KEY',    endpoint:'https://generativelanguage.googleapis.com',     extra:[]},
  {name:'azure_openai',    key:'AZURE_OPENAI_KEY',  endpoint:'见 AZURE_OPENAI_ENDPOINT',                     extra:['AZURE_OPENAI_ENDPOINT']},
];
console.log('=== image-gen 配置状态 ===\n');
for (const p of providers) {
  const ok = !!process.env[p.key];
  console.log(`[${p.name}] ${ok?'✅ 已配置':'❌ 未配置'}`);
  console.log(`  Key: ${p.key} | Endpoint: ${p.endpoint}`);
  for (const e of p.extra) console.log(`  ${e}: ${process.env[e]?'✅':'❌'}`);
  console.log('');
}
console.log('提示: 在 ~/.claude/settings.json 的 env 字段同时配置多个 Key，用 --provider 切换。');
EOF
```

---

## 错误处理

| 情况 | 处理 |
|------|------|
| 所有 Key 均未配置 | 展示四个供应商的配置方法，退出 |
| 指定 provider 的 Key 未设置 | 提示缺少的变量名，建议切换到已配置的供应商 |
| volcengine_maas 尺寸 < 2048x2048 | 自动调整为 `2048x2048` 并告知 |
| API 返回错误 | 原文输出，不猜测原因 |
| 保存目录不存在 | 自动创建 |
| 用户描述过于模糊 | 生成提示词后展示，询问是否满意 |
| `--from` 路径不存在 | 提示文件不存在，询问正确路径 |
