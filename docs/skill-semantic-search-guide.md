# OpenClaw Skill 语义检索：复现指南

> 本文档目标：让任何 OpenClaw 用户都能从零复现 Skill 语义检索能力。
> 将本文档发给 OpenClaw（AI 助手），它可以全程自动完成部署。

---

## 背景

OpenClaw 默认会把所有已安装 Skill 的 name + description 完整注入到每轮对话的 system prompt 里。
当 Skill 数量较多时（如 59 个，约 4,867 tokens/轮），会带来以下问题：

- **token 浪费**：每轮对话固定消耗大量 token 在 Skill 列表上
- **上下文污染**：无关信息干扰 LLM 对工具的判断
- **成本放大**：高频调用场景下累计成本显著增加

**语义检索方案**：每次用户消息到达时，先用向量检索找出最相关的 Top-K 个 Skill，
通过动态过滤大幅减少注入 system prompt 的 Skill 数量，从而降低 token 消耗。
整个过程无需修改 OpenClaw 核心代码。

---

## 技术架构

```
用户消息到达
    ↓
[message:received hook] skill-router 触发
    ↓
Python 调用 OpenAI Embedding API 向量化用户查询
    ↓
COS Vectors 检索 Top-5 相关 Skills（cosine 距离）
    ↓
写入 openclaw.json agents.<name>.skills = [Top-5 names]
    ↓
SIGUSR1 热重载 Gateway config
    ↓
LLM 收到消息时，system prompt 只包含 Top-5 精选 Skills
```

### Embedding 接入方式

本方案通过 **OpenAI 兼容的 Embedding API** 获取向量，支持两种模式：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **OpenAI API**（推荐） | 直接调用 OpenAI `text-embedding-3-small`，无需本地模型 | 正式使用、追求效果和稳定性 |
| **本地兼容服务** | Ollama 等本地部署，提供 OpenAI 兼容的 `/v1/embeddings` 接口 | 新手体验、离线环境、数据隐私要求高 |

> 由于两种模式都遵循 OpenAI API 协议，脚本代码完全一致，只需修改 `EMBEDDING_BASE_URL`、`EMBEDDING_API_KEY`、`EMBEDDING_MODEL` 三个配置即可切换。

### 核心组件

| 组件 | 说明 |
|------|------|
| **Embedding 接口** | OpenAI 兼容的 `/v1/embeddings` API（OpenAI 官方或本地兼容服务） |
| **向量存储与检索** | 腾讯云 COS Vectors（通过 cos-vectors-skill 操作） |
| **cos-vectors-skill** | 封装 COS Vectors 完整 CRUD，提供 16 个操作脚本 |
| **索引构建脚本** | `build_skill_index.py`（离线，将所有 SKILL.md 向量化并写入向量桶） |
| **在线检索脚本** | `retrieve_skills.py`（每次消息触发，查询 Top-K 相关 Skill） |
| **Hook** | `skill-router`（监听 `message:received`，检索 → 写 config → 热重载） |
| **配置路径** | `/projects/.openclaw/openclaw.json` |

### COS Vectors 索引参数

| 参数 | 值 |
|------|-----|
| 维度 | 1536（对应 `text-embedding-3-small` 输出；本地模型按实际维度填写） |
| 距离度量 | cosine |
| 距离阈值 | < 0.72（超过则认为无相关 Skill） |

---

## 部署步骤

### 第一步：安装 cos-vectors-skill

cos-vectors-skill 封装了 COS Vectors 的完整操作能力，包括向量桶/索引管理、向量数据 CRUD 等。
本方案所有与 COS Vectors 的交互（创建索引、写入向量、查询向量）均通过此 skill 完成。

从clawhub安装，skill链接地址是： https://clawhub.ai/jimmyyan/cos-vectors-skill
如果失败，尝试从skillhub安装，先安装skillhub: curl -fsSL https://skillhub-1388575217.cos.ap-guangzhou.myqcloud.com/install/install.sh | bash
再安装cos-vectors-skill: skillhub install cos-vectors-skill

安装完成后，在 OpenClaw 中确认 skill 可用：

```bash
openclaw skills list | grep cos-vector
```

> **说明**：cos-vectors-skill 提供 16 个操作脚本，覆盖向量桶、索引、向量数据的完整 CRUD，
> 以及查询、过滤、批量操作等能力。后续步骤中的创建索引和查询操作均基于此 skill。

### 第二步：准备 COS 向量桶并创建索引

配置 cos-vectors-skill 所需的 COS 凭据（SecretId / SecretKey）。
然后通过 cos-vectors-skill 创建向量索引：

1. 在 OpenClaw 中对话，要求它用 cos-vectors-skill 创建索引：

   > 请用 cos-vectors-skill 在 bucket `<your-bucket>` 中创建一个向量索引，
   > 索引名 `skill-index`，维度 1536，距离度量 cosine。

   **注意**：维度必须与你使用的 Embedding 模型输出一致：
   - OpenAI `text-embedding-3-small` → **1536**
   - Ollama `nomic-embed-text` → **768**
   - 其他模型请查看对应文档

2. 记录 Bucket 名称（含 APPID 后缀）、Region 和 Index 名。

> **前置条件**：需要一个已开通向量功能的 COS Bucket。
> 如还未创建，登录[腾讯云控制台](https://console.cloud.tencent.com/cos)新建 Bucket 并开通向量检索功能。

### 第三步：安装 Python 依赖

```bash
pip3 install openai cos-python-sdk-v5 pyyaml
```

仅需 3 个包：
- `openai`：调用 OpenAI 兼容的 Embedding API
- `cos-python-sdk-v5`：操作 COS Vectors
- `pyyaml`：解析 SKILL.md 的 YAML frontmatter

### 第四步：部署脚本

#### 4.1 创建目录

```bash
mkdir -p ~/workspace/scripts
```

#### 4.2 离线构建脚本 `build_skill_index.py`

保存为 `~/workspace/scripts/build_skill_index.py`。

此脚本通过 OpenAI Embedding API 向量化所有 Skill，再通过 COS Vectors API 写入向量桶：

```python
#!/usr/bin/env python3
"""
build_skill_index.py — 离线构建 Skill 向量索引

扫描所有 SKILL.md，通过 OpenAI Embedding API 向量化后写入 COS Vectors。
每次安装新 Skill 后重跑此脚本即可增量更新。

用法：python3 build_skill_index.py
"""

import os
import re
import sys
import pathlib
import yaml
from openai import OpenAI
from qcloud_cos import CosConfig, CosVectorsClient

# ── Embedding API 配置（修改这里）────────────────────────────────────────────
# 方式一：使用 OpenAI 官方 API
EMBEDDING_BASE_URL = "https://api.openai.com/v1"
EMBEDDING_API_KEY  = "YOUR_OPENAI_API_KEY"
EMBEDDING_MODEL    = "text-embedding-3-small"    # 输出维度 1536

# 方式二：使用本地 Ollama（取消下面三行注释，注释掉上面三行）
# EMBEDDING_BASE_URL = "http://localhost:11434/v1"
# EMBEDDING_API_KEY  = "ollama"                   # Ollama 不校验，但必须填
# EMBEDDING_MODEL    = "nomic-embed-text"         # 输出维度 768

# ── COS Vectors 配置（修改这里）──────────────────────────────────────────────
SECRET_ID    = "YOUR_SECRET_ID"
SECRET_KEY   = "YOUR_SECRET_KEY"
BUCKET       = "your-bucket-1234567890"
INDEX        = "skill-index"
REGION       = "ap-guangzhou"
DOMAIN       = f"vectors.{REGION}.coslake.com"

# Skill 扫描目录
SKILL_DIRS = [
    pathlib.Path("/usr/local/lib/.nvm/versions/node/v22.17.0/lib/node_modules/openclaw/skills"),
    pathlib.Path("/projects/.openclaw/skills"),
]

# ── Embedding 客户端 ─────────────────────────────────────────────────────────
embed_client = OpenAI(base_url=EMBEDDING_BASE_URL, api_key=EMBEDDING_API_KEY)

def get_embeddings(texts: list[str]) -> list[list[float]]:
    """批量获取文本向量（自动分批，OpenAI 单次最多 2048 条）"""
    all_embeddings = []
    BATCH = 512  # 每批发送数量，保守值
    for i in range(0, len(texts), BATCH):
        batch = texts[i:i + BATCH]
        resp = embed_client.embeddings.create(input=batch, model=EMBEDDING_MODEL)
        # 按 index 排序确保顺序一致
        sorted_data = sorted(resp.data, key=lambda x: x.index)
        all_embeddings.extend([d.embedding for d in sorted_data])
    return all_embeddings

# ── COS Vectors 客户端 ────────────────────────────────────────────────────────
_cfg    = CosConfig(Region=REGION, SecretId=SECRET_ID, SecretKey=SECRET_KEY,
                    Scheme="https", Domain=DOMAIN)
vclient = CosVectorsClient(_cfg)

# ── 收集所有 SKILL.md ─────────────────────────────────────────────────────────
def collect_skills():
    skills = []
    for base in SKILL_DIRS:
        if not base.exists():
            continue
        for skill_md in sorted(base.rglob("SKILL.md")):
            name    = skill_md.parent.name
            content = skill_md.read_text(encoding="utf-8")
            # 提取 YAML frontmatter 里的 description（优先）
            description = ""
            if content.startswith("---"):
                end = content.find("---", 3)
                if end != -1:
                    try:
                        meta = yaml.safe_load(content[3:end])
                        description = meta.get("description", "") or ""
                    except Exception:
                        pass
            text = f"{name}\n{description}" if description else name
            skills.append({"name": name, "text": text})
    return skills

# ── 向量化并写入 ──────────────────────────────────────────────────────────────
print("扫描 SKILL.md...")
skills = collect_skills()
print(f"找到 {len(skills)} 个 skill")

texts = [s["text"] for s in skills]
print(f"调用 Embedding API（{EMBEDDING_MODEL}）向量化中...")
embeddings = get_embeddings(texts)
print(f"向量化完成，维度: {len(embeddings[0])}")

vectors = []
for i, sk in enumerate(skills):
    key = re.sub(r"[^a-zA-Z0-9\-_.]", "_", f"skill:{sk['name']}")[:100]
    vectors.append({
        "key":      key,
        "data":     {"float32": embeddings[i]},
        "metadata": {"name": sk["name"]},
    })

# 分批写入（每批最多 100 条）
BATCH = 100
for i in range(0, len(vectors), BATCH):
    batch = vectors[i:i + BATCH]
    end   = min(i + BATCH, len(vectors))
    print(f"写入向量 {i+1}~{end}/{len(vectors)}...", end="", flush=True)
    vclient.put_vectors(Bucket=BUCKET, Index=INDEX, Vectors=batch)
    print(" ✓")

print(f"\n✅ 全部完成！共上传 {len(vectors)} 个 skill 向量到 {BUCKET}/{INDEX}")
```

#### 4.3 在线检索脚本 `retrieve_skills.py`

保存为 `~/workspace/scripts/retrieve_skills.py`。

此脚本调用 OpenAI Embedding API 向量化查询，再用 COS Vectors `query_vectors` 完成检索：

```python
#!/usr/bin/env python3
"""
retrieve_skills.py — 在线检索相关 Skills

通过 OpenAI 兼容 Embedding API 向量化查询，使用 COS Vectors 进行语义检索。

用法：python3 retrieve_skills.py "用户查询" [--top-k=5] [--json]
"""

import sys
import json
from openai import OpenAI
from qcloud_cos import CosConfig, CosVectorsClient

# ── Embedding API 配置（修改这里）────────────────────────────────────────────
# 方式一：使用 OpenAI 官方 API
EMBEDDING_BASE_URL = "https://api.openai.com/v1"
EMBEDDING_API_KEY  = "YOUR_OPENAI_API_KEY"
EMBEDDING_MODEL    = "text-embedding-3-small"    # 输出维度 1536

# 方式二：使用本地 Ollama（取消下面三行注释，注释掉上面三行）
# EMBEDDING_BASE_URL = "http://localhost:11434/v1"
# EMBEDDING_API_KEY  = "ollama"                   # Ollama 不校验，但必须填
# EMBEDDING_MODEL    = "nomic-embed-text"         # 输出维度 768

# ── COS Vectors 配置（修改这里）──────────────────────────────────────────────
SECRET_ID  = "YOUR_SECRET_ID"
SECRET_KEY = "YOUR_SECRET_KEY"
BUCKET     = "your-bucket-1234567890"
INDEX      = "skill-index"
REGION     = "ap-guangzhou"
DOMAIN     = f"vectors.{REGION}.coslake.com"

DISTANCE_THRESHOLD = 0.72   # 超过此值则认为无相关 Skill

# ── Embedding 客户端（单例）────────────────────────────────────────────────
_embed_client = None

def get_embed_client():
    global _embed_client
    if _embed_client is None:
        _embed_client = OpenAI(base_url=EMBEDDING_BASE_URL, api_key=EMBEDDING_API_KEY)
    return _embed_client

def get_embedding(text: str) -> list[float]:
    """获取单条文本的向量"""
    client = get_embed_client()
    resp = client.embeddings.create(input=[text], model=EMBEDDING_MODEL)
    return resp.data[0].embedding

# ── COS Vectors 客户端 ────────────────────────────────────────────────────────
_cfg    = CosConfig(Region=REGION, SecretId=SECRET_ID, SecretKey=SECRET_KEY,
                    Scheme="https", Domain=DOMAIN)
vclient = CosVectorsClient(_cfg)

# ── 检索函数 ─────────────────────────────────────────────────────────────────
def retrieve_skills(query: str, top_k: int = 5) -> list:
    vec = get_embedding(query)

    _, data = vclient.query_vectors(
        Bucket=BUCKET, Index=INDEX,
        QueryVector={"float32": vec},
        TopK=top_k,
        ReturnDistance=True,
        ReturnMetaData=True,
    )

    results = []
    for item in data.get("vectors", []):
        meta     = item.get("metadata") or {}
        name     = meta.get("name") or item.get("key", "")
        distance = round(item.get("distance", 1.0), 4)
        if distance > DISTANCE_THRESHOLD:
            continue
        results.append({"name": name, "distance": distance})

    return results

# ── 入口 ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    args    = sys.argv[1:]
    query   = next((a for a in args if not a.startswith("--")), "")
    top_k   = int(next((a.split("=")[1] for a in args if a.startswith("--top-k=")), "5"))
    as_json = "--json" in args

    if not query:
        print("用法: python3 retrieve_skills.py \"查询内容\" [--top-k=5] [--json]",
              file=sys.stderr)
        sys.exit(1)

    results = retrieve_skills(query, top_k)

    if as_json:
        print(json.dumps(results, ensure_ascii=False))
    else:
        if not results:
            print(f"未找到相关 Skill（所有距离 > {DISTANCE_THRESHOLD}）")
        else:
            for r in results:
                print(f"  {r['name']:<30} distance={r['distance']}")
```

### 第五步：修改脚本配置

打开两个脚本，替换顶部配置：

```python
# ── Embedding API 配置 ───────────────────────────────────────────────────────
EMBEDDING_BASE_URL = "https://api.openai.com/v1"    # OpenAI 官方；或本地服务地址
EMBEDDING_API_KEY  = "YOUR_OPENAI_API_KEY"           # OpenAI API Key
EMBEDDING_MODEL    = "text-embedding-3-small"        # 模型名

# ── COS Vectors 配置 ────────────────────────────────────────────────────────
SECRET_ID  = "YOUR_SECRET_ID"           # 你的腾讯云 SecretId
SECRET_KEY = "YOUR_SECRET_KEY"          # 你的腾讯云 SecretKey
BUCKET     = "your-bucket-1234567890"   # COS Bucket 名称（含 APPID 后缀）
REGION     = "ap-guangzhou"             # Bucket 所在地域
INDEX      = "skill-index"             # 第二步创建的 Index 名
```

### 第六步：构建向量索引

```bash
python3 ~/workspace/scripts/build_skill_index.py
```

预期输出：
```
扫描 SKILL.md...
找到 59 个 skill
调用 Embedding API（text-embedding-3-small）向量化中...
向量化完成，维度: 1536
写入向量 1~59/59... ✓

✅ 全部完成！共上传 59 个 skill 向量到 your-bucket-1234567890/skill-index
```

验证检索效果：
```bash
python3 ~/workspace/scripts/retrieve_skills.py "帮我查代码仓库"
# 预期：github  distance=0.21 ...

python3 ~/workspace/scripts/retrieve_skills.py "今天深圳天气怎么样"
# 预期：weather  distance=0.18 ...
```

也可以通过 cos-vectors-skill 查询已写入的向量数量来验证索引状态：

> 请用 cos-vectors-skill 列出 bucket `<your-bucket>` 中 `skill-index` 索引的向量数量。

### 第七步：编写 skill-router Hook

#### 7.1 Hook 目录结构

```
~/workspace/hooks/skill-router/
├── HOOK.md       # 声明监听的事件
└── handler.ts    # 检索 → 写 config → 热重载
```

#### 7.2 HOOK.md

保存为 `~/workspace/hooks/skill-router/HOOK.md`：

```markdown
---
name: skill-router
description: 在每条消息处理前，语义检索最相关的 Skills 并动态更新 Gateway config
events:
  - message:received
---
```

#### 7.3 handler.ts

保存为 `~/workspace/hooks/skill-router/handler.ts`：

```typescript
import { execSync } from "child_process";
import { readFileSync, writeFileSync } from "fs";

// ── 配置 ──────────────────────────────────────────────────────────────────────
const RETRIEVE_SCRIPT = `${process.env.HOME}/workspace/scripts/retrieve_skills.py`;
const CONFIG_PATH     = "/projects/.openclaw/openclaw.json";
const AGENT_NAME      = "marcus";       // 替换为你的 Agent 名称
const TOP_K           = 5;
const DISTANCE_THRESHOLD = 0.72;
const DEBOUNCE_MS     = 3000;           // 3 秒内同一会话不重复检索

// ── 防抖状态 ──────────────────────────────────────────────────────────────────
const lastRetrieveTime: Record<string, number> = {};

// ── Hook 处理函数 ─────────────────────────────────────────────────────────────
export const handler = async (event: any) => {
  if (event.type !== "message" || event.action !== "received") return;

  const content        = event.context?.content ?? "";
  const conversationId = event.context?.conversationId ?? "default";
  if (!content) return;

  // 防抖
  const now = Date.now();
  if (now - (lastRetrieveTime[conversationId] ?? 0) < DEBOUNCE_MS) return;
  lastRetrieveTime[conversationId] = now;

  try {
    // 1. 向量检索（调用 retrieve_skills.py，底层通过 OpenAI Embedding API 获取向量）
    const raw     = execSync(
      `python3 ${RETRIEVE_SCRIPT} ${JSON.stringify(content)} --top-k=${TOP_K} --json`,
      { timeout: 15000 }
    ).toString();
    const skills: Array<{ name: string; distance: number }> = JSON.parse(raw);
    const relevant = skills
      .filter(s => s.distance < DISTANCE_THRESHOLD)
      .map(s => s.name);

    if (relevant.length === 0) return;  // 无相关 Skill，不修改 config

    // 2. 读取并更新 openclaw.json
    const cfg    = JSON.parse(readFileSync(CONFIG_PATH, "utf-8"));
    const agents = cfg?.agents?.list ?? [];
    const idx    = agents.findIndex((a: any) => a.name === AGENT_NAME);
    if (idx === -1) {
      console.error(`[skill-router] 找不到 Agent: ${AGENT_NAME}`);
      return;
    }
    agents[idx].skills = relevant;
    writeFileSync(CONFIG_PATH, JSON.stringify(cfg, null, 2));

    // 3. 热重载 Gateway（SIGUSR1）
    execSync("kill -USR1 $(pgrep -f 'openclaw-gateway' | head -1)");

    console.log(`[skill-router] 已更新 Skills: [${relevant.join(", ")}]`);
  } catch (err) {
    console.error("[skill-router] 检索失败:", err);
  }
};
```

> **注意**：将 `AGENT_NAME` 替换为你在 `openclaw.json` 中配置的 Agent 名称。

### 第八步：注册并启用 Hook

在 OpenClaw 配置文件（`/projects/.openclaw/openclaw.json`）中，为 hooks 添加自定义目录：

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["~/workspace/hooks"]
      }
    }
  }
}
```

然后启用 Hook 并重启：

```bash
openclaw hooks enable skill-router
# 预期：✓ Enabled hook: 🧭 skill-router

openclaw hooks list

openclaw gateway restart
```

---

## 验证部署

发送一条测试消息，观察 Gateway 日志中是否出现：

```
[skill-router] 已更新 Skills: [github, weather, ...]
```

也可以用 cos-vectors-skill 直接查询向量桶，确认向量数据完整：

> 请用 cos-vectors-skill 查询 `skill-index` 中与"帮我查代码"最相近的 3 个向量。

---

## 附录 A：新手本地快速体验（无需 OpenAI API Key）

> 如果你没有 OpenAI API Key，或者希望完全离线运行、不产生任何外部 API 费用，
> 可以在本地部署一个兼容 OpenAI 的 Embedding 服务。以下提供两种方案。

### 方案一：Ollama（推荐，最简单）

[Ollama](https://ollama.com) 是一个轻量级本地模型运行框架，原生支持 OpenAI 兼容 API。
安装后只需两条命令即可启动 Embedding 服务。

#### 1. 安装 Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# macOS 也可以用 Homebrew
brew install ollama
```

Windows 用户前往 [ollama.com/download](https://ollama.com/download) 下载安装包。

#### 2. 拉取 Embedding 模型

```bash
ollama pull nomic-embed-text
```

`nomic-embed-text` 是一个高质量的通用 Embedding 模型，支持中英文，维度 768，模型仅 ~274MB。

> 其他可选模型：
> - `mxbai-embed-large`（1024 维，效果更好但更大）
> - `all-minilm`（384 维，最小最快）

#### 3. 启动服务

```bash
ollama serve
# 默认监听 http://localhost:11434
```

> Ollama 安装后通常会自动作为后台服务运行，可以跳过此步。
> 验证：`curl http://localhost:11434` 应返回 "Ollama is running"。

#### 4. 验证 Embedding API

```bash
curl http://localhost:11434/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nomic-embed-text",
    "input": "测试向量化"
  }'
```

应返回包含 `embedding` 数组的 JSON 响应。

#### 5. 修改脚本配置

在 `build_skill_index.py` 和 `retrieve_skills.py` 中，将 Embedding 配置改为：

```python
EMBEDDING_BASE_URL = "http://localhost:11434/v1"
EMBEDDING_API_KEY  = "ollama"              # Ollama 不校验，但 openai 库要求必填
EMBEDDING_MODEL    = "nomic-embed-text"    # 输出维度 768
```

同时，**COS 向量索引维度也要改为 768**（与模型输出一致）。

### 方案二：用 FastAPI 自建微服务（更灵活）

如果你想用 `sentence-transformers` 的任意模型，但又希望通过 OpenAI 兼容 API 暴露出来，
可以用几十行 Python 代码搭建一个微服务。

#### 1. 安装依赖

```bash
pip3 install fastapi uvicorn "sentence-transformers[onnx]"
```

#### 2. 创建服务文件

保存为 `~/workspace/scripts/embedding_server.py`：

```python
#!/usr/bin/env python3
"""
embedding_server.py — 本地 OpenAI 兼容 Embedding 服务

基于 sentence-transformers，提供 /v1/embeddings 接口，
请求/响应格式完全兼容 OpenAI Embedding API。

启动：uvicorn embedding_server:app --host 0.0.0.0 --port 8686
"""

import time
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pydantic import BaseModel
from sentence_transformers import SentenceTransformer

# ── 配置 ──────────────────────────────────────────────────────────────────────
MODEL_NAME = "shibing624/text2vec-base-chinese"  # 可换成任意 sentence-transformers 模型
# 使用 ONNX 加速（可选，去掉 backend 和 model_kwargs 则用 PyTorch）
USE_ONNX   = True

# ── 模型加载 ──────────────────────────────────────────────────────────────────
model = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global model
    print(f"加载模型: {MODEL_NAME} ...")
    kwargs = {}
    if USE_ONNX:
        kwargs = {"backend": "onnx", "model_kwargs": {"file_name": "onnx/model_qint8_avx512_vnni.onnx"}}
    model = SentenceTransformer(MODEL_NAME, **kwargs)
    print(f"模型就绪，维度: {model.get_sentence_embedding_dimension()}")
    yield

app = FastAPI(title="Local Embedding Server", lifespan=lifespan)

# ── 请求/响应模型（兼容 OpenAI 格式）────────────────────────────────────────
class EmbeddingRequest(BaseModel):
    input: str | list[str]
    model: str = MODEL_NAME
    encoding_format: str = "float"

class EmbeddingData(BaseModel):
    object: str = "embedding"
    embedding: list[float]
    index: int

class EmbeddingResponse(BaseModel):
    object: str = "list"
    data: list[EmbeddingData]
    model: str
    usage: dict

# ── API 端点 ──────────────────────────────────────────────────────────────────
@app.post("/v1/embeddings")
async def create_embeddings(req: EmbeddingRequest) -> EmbeddingResponse:
    texts = [req.input] if isinstance(req.input, str) else req.input
    embeddings = model.encode(texts, normalize_embeddings=True)

    data = [
        EmbeddingData(embedding=emb.tolist(), index=i)
        for i, emb in enumerate(embeddings)
    ]

    return EmbeddingResponse(
        data=data,
        model=req.model,
        usage={"prompt_tokens": sum(len(t) for t in texts), "total_tokens": sum(len(t) for t in texts)},
    )

# ── 健康检查 ──────────────────────────────────────────────────────────────────
@app.get("/")
async def health():
    return {"status": "ok", "model": MODEL_NAME}
```

#### 3. 启动服务

```bash
# 首次启动会自动下载模型（约 200MB），后续启动秒开
uvicorn embedding_server:app --host 0.0.0.0 --port 8686
```

#### 4. 验证

```bash
curl http://localhost:8686/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input": "测试向量化", "model": "text2vec"}'
```

#### 5. 修改脚本配置

```python
EMBEDDING_BASE_URL = "http://localhost:8686/v1"
EMBEDDING_API_KEY  = "not-needed"          # 本地服务不校验
EMBEDDING_MODEL    = "text2vec"            # 任意值，服务端会忽略
```

同时 COS 向量索引维度改为模型对应值（`text2vec-base-chinese` → **768**）。

### 方案对比

| 维度 | Ollama | FastAPI 自建 |
|------|--------|-------------|
| **安装复杂度** | 极低（两条命令） | 中等（需写代码） |
| **模型选择** | Ollama 模型库中的 Embedding 模型 | 任意 sentence-transformers 模型 |
| **性能** | 优秀（原生优化） | 取决于模型和后端（ONNX 较快） |
| **灵活性** | 中等 | 极高（完全可控） |
| **适合场景** | 新手快速体验、日常使用 | 需要特定模型、深度定制 |

---

## 日常维护

### 新装 Skill 后更新索引

```bash
python3 ~/workspace/scripts/build_skill_index.py
```

新 Skill 会被增量写入向量桶（key 不重复则自动覆盖更新）。

### 手动测试检索效果

```bash
python3 ~/workspace/scripts/retrieve_skills.py "你的查询" --top-k=5
```

### 通过 cos-vectors-skill 查看索引状态

> 请用 cos-vectors-skill 列出 `<your-bucket>` 中 `skill-index` 索引的所有向量 key。

### 清空并重建索引

> 请用 cos-vectors-skill 删除 `<your-bucket>` 中 `skill-index` 的所有向量。

然后重跑构建脚本：

```bash
python3 ~/workspace/scripts/build_skill_index.py
```

### 调整距离阈值

在 `handler.ts` 和 `retrieve_skills.py` 中同时修改 `DISTANCE_THRESHOLD`，然后重启 Gateway：

```bash
openclaw gateway restart
```

### 切换 Embedding 提供方

只需修改两个脚本中的 3 个配置项，**无需改动任何业务逻辑代码**：

```python
EMBEDDING_BASE_URL = "..."   # API 地址
EMBEDDING_API_KEY  = "..."   # API Key
EMBEDDING_MODEL    = "..."   # 模型名
```

> **注意**：切换模型后向量维度可能变化，需要重新创建 COS 向量索引（维度要匹配），
> 并重跑 `build_skill_index.py` 重建所有向量。

### 关键文件路径

| 文件 | 路径 |
|------|------|
| cos-vectors-skill | `/usr/local/lib/.nvm/.../openclaw/skills/tencent/cos-vector-skill/` |
| Hook 目录 | `~/workspace/hooks/skill-router/` |
| 离线构建脚本 | `~/workspace/scripts/build_skill_index.py` |
| 在线检索脚本 | `~/workspace/scripts/retrieve_skills.py` |
| 本地 Embedding 服务（可选） | `~/workspace/scripts/embedding_server.py` |
| OpenClaw 配置 | `/projects/.openclaw/openclaw.json` |

---

## 让 OpenClaw 自动完成部署（一键指令）

将以下内容发给 OpenClaw，填入你自己的参数，它会全程自动搞定：

---

> 请帮我部署 Skill 语义检索功能（skill-router），完整步骤如下：
>
> **参数配置：**
> - OpenAI API Key：`<你的 OpenAI API Key>`（或本地服务地址）
> - Embedding 模型：`text-embedding-3-small`（或本地模型名）
> - 腾讯云 SecretId：`<你的SecretId>`
> - 腾讯云 SecretKey：`<你的SecretKey>`
> - COS Bucket 名称：`<your-bucket-1234567890>`
> - COS Region：`<ap-guangzhou>`
> - 向量 Index 名：`skill-index`
> - 向量维度：`1536`（text-embedding-3-small）或 `768`（nomic-embed-text / text2vec）
> - Agent 名称：`<你的Agent名称>`（即 openclaw.json 中 agents.list 里的 name）
>
> **请按顺序执行：**
> 1. 安装 cos-vectors-skill：从 `https://clawhub.ai/jimmyyan/cos-vectors-skill` 安装，如果失败：尝试从skillhub安装，先安装skillhub: curl -fsSL https://skillhub-1388575217.cos.ap-guangzhou.myqcloud.com/install/install.sh | bash，再安装cos-vectors-skill: skillhub install cos-vectors-skill
> 2. 用 cos-vectors-skill 在指定 Bucket 创建向量索引 `skill-index`（维度按模型填写，cosine 距离）
> 3. 运行 `pip3 install openai cos-python-sdk-v5 pyyaml` 安装依赖
> 4. 创建目录 `~/workspace/scripts/` 和 `~/workspace/hooks/skill-router/`
> 5. 将 `build_skill_index.py`、`retrieve_skills.py`、`HOOK.md`、`handler.ts` 按文档保存到对应路径，替换配置参数
> 6. 运行 `build_skill_index.py` 建立向量索引；用 cos-vectors-skill 验证向量数量
> 7. 在 `openclaw.json` 的 hooks.internal.load.extraDirs 中添加 `~/workspace/hooks`
> 8. 运行 `openclaw hooks enable skill-router`，然后 `openclaw gateway restart`
> 9. 验证：运行 `retrieve_skills.py "查天气"` 确认返回合理结果
>
> 脚本内容参考本文档中的完整代码。

---

## 常见问题

**Q：向量维度应该填多少？**
A：取决于你使用的 Embedding 模型。常见值：
- OpenAI `text-embedding-3-small` → **1536**
- OpenAI `text-embedding-3-large` → **3072**
- Ollama `nomic-embed-text` → **768**
- `text2vec-base-chinese` → **768**
创建 COS 向量索引时的维度必须与模型输出一致。

**Q：cos-vectors-skill 和 build_skill_index.py 的关系是什么？**
A：两者使用完全相同的 COS Vectors API（`put_vectors`、`query_vectors` 等）。
cos-vectors-skill 封装了对话式操作入口，方便在 OpenClaw 中通过自然语言管理向量桶；
`build_skill_index.py` 是批量写入的离线脚本，直接调用相同的底层 SDK。

**Q：使用 OpenAI API 的成本如何？**
A：`text-embedding-3-small` 的价格为 $0.02 / 1M tokens。
59 个 Skill 构建索引约消耗 ~2K tokens（约 $0.00004），几乎可以忽略。
每次用户消息检索约消耗 ~20 tokens（约 $0.0000004），极其便宜。

**Q：可以用 OpenAI 兼容的第三方服务吗？**
A：可以。任何提供 `/v1/embeddings` 接口且兼容 OpenAI 格式的服务都行，
例如 Azure OpenAI、各大云厂商的 Embedding API、本地 Ollama 等。
只需修改 `EMBEDDING_BASE_URL`、`EMBEDDING_API_KEY`、`EMBEDDING_MODEL` 即可。

**Q：切换模型后需要重建索引吗？**
A：是的。不同模型的向量空间不同，切换后必须：
1. 创建新的 COS 向量索引（维度匹配新模型）
2. 重跑 `build_skill_index.py` 重建所有向量
3. 更新脚本中的 INDEX 名称

**Q：distance 阈值 0.72 怎么来的？**
A：实测纯聊天类查询（如"你好"）命中最近 Skill 的距离约为 0.64，工具类查询通常 < 0.5。0.72 是一个较宽松的边界，实际使用中可根据效果下调至 0.65 左右。
注意：不同 Embedding 模型的距离分布可能不同，切换模型后建议重新测试并调整阈值。

**Q：安装新 Skill 后检索不到？**
A：重跑 `build_skill_index.py` 即可，新 Skill 会被增量写入。
也可以用 cos-vectors-skill 确认新 Skill 的向量是否已存在于索引中。

**Q：Hook 修改 config 后需要手动重载吗？**
A：不需要。`handler.ts` 中的 `kill -USR1` 会触发 OpenClaw Gateway 的 SIGUSR1 热重载，自动生效。

**Q：本地 Ollama 服务的延迟如何？**
A：`nomic-embed-text` 单条文本 Embedding 约 50~200ms（取决于硬件），
59 条批量构建约 3~5 秒。在线检索单次约 100~300ms，完全满足实时性要求。

---

## 参考资料

- [OpenAI Embeddings API 文档](https://platform.openai.com/docs/guides/embeddings)
- [OpenAI Python SDK](https://github.com/openai/openai-python)
- [Ollama 官网](https://ollama.com)
- [Ollama OpenAI 兼容文档](https://docs.ollama.com/api/openai-compatibility)
- [cos-vectors-skill（ClawHub）](https://clawhub.ai/jimmyyan/cos-vectors-skill)
- [COS 向量检索文档](https://cloud.tencent.com/document/product/436/100411)
- [cos-python-sdk-v5](https://github.com/tencentyun/cos-python-sdk-v5)
- [sentence-transformers 文档](https://sbert.net/)
- [OpenClaw Hooks 文档](https://docs.openclaw.ai)