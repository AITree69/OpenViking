# OpenViking + OpenClaw 多模态部署指南

> **适用版本**：OpenViking `>= 2026.5` / OpenClaw `>= 2026.5.27` / `@openviking/openclaw-plugin >= 2026.7.x`
>
> **读者**：从零开始把 OpenViking + OpenClaw 部署起来并打开多模态能力（图像 / 音频 / 视频 / PDF）的开发人员 / 运维。
>
> **预计耗时**：30 - 60 分钟（含申请 API key、装包、改配置、跑通一个端到端测试）。

---

## 0. 一句话总结

**OpenClaw 客户端本身不解析图像 / 音频 / 视频**，所有多模态都外包给 OpenViking 服务的 parser 模块。所以"打开多模态" = **让 OpenViking 服务能调到 VLM + 多模态 embedding**，然后**让 OpenClaw agent 能调 `add_resource` 工具把多模态文件喂给 OpenViking**。两件事必须同时做完，缺一件就只会"看着像开了，实际还是文字 only"。

---

## 1. 架构与责任分工

```
┌────────────────────────────────────────────────────────────────────┐
│  OpenClaw 客户端 (Node.js)                                          │
│  ────────────────────────────                                      │
│  • 14 个内置工具全部是文本/检索类：                                  │
│      add_skill / ov_search / ov_read / ov_list / memory_recall …   │
│  • 多模态文件需要通过 add_resource 工具转交给 OpenViking            │
│  • add_resource 默认 disabled，需要显式 enableAddResourceTool=true  │
└─────────────────────────────────┬──────────────────────────────────┘
                                  │ HTTP (baseUrl: http://127.0.0.1:1933)
                                  ▼
┌────────────────────────────────────────────────────────────────────┐
│  OpenViking 服务 (Python)                                           │
│  ──────────────────────────                                         │
│  • 接收 add_resource 上传的任意文件                                  │
│  • 根据 ov.conf.parsers.* 选择 parser：                              │
│      image → ImageParser (VLM + 可选 OCR)                           │
│      audio → AudioParser (Whisper)                                  │
│      video → VideoParser (抽帧 + VLM 描述 + Whisper)                 │
│      pdf   → PDFParser (mineru 外部接口)                             │
│  • 调用 ov.conf.vlm 描述图像/视频帧                                 │
│  • 调用 ov.conf.embedding.dense 向量化（multimodal 模式同时吃图像）  │
│  • 写入本地 AGFS + 本地/远程向量库                                   │
└─────────────────────────────────┬──────────────────────────────────┘
                                  │ HTTPS
                                  ▼
┌────────────────────────────────────────────────────────────────────┐
│  外部模型服务 (火山方舟 Ark / OpenAI / Ollama / Kimi / GLM …)       │
│  ──────────────────────────────────────────────────                │
│  • VLM: 视觉理解（doubao-seed-2-0-lite / gpt-4-vision / glm-4.6v）  │
│  • Embedding: 多模态向量（doubao-embedding-vision / nomic …）       │
│  • Audio ASR: whisper-large-v3 / 火山 ASR                           │
│  • PDF: mineru 私有部署                                              │
└────────────────────────────────────────────────────────────────────┘
```

> **关键认识**：`OpenViking` 仓**不带**任何 VLM / embedding 服务的 API key，也不内置模型；所有调用都靠用户在 `ov.conf` 里配外部 endpoint。同理 `OpenClaw` 仓**不带**图像/音频/视频处理能力，所有多模态要靠 `add_resource` 把任务转给 OpenViking。

---

## 2. 前置条件

| 项 | 要求 |
| --- | --- |
| Python | `>= 3.10` |
| Node.js | `>= 22`（OpenClaw 2026.5.27+ 的硬性要求） |
| OpenClaw | `>= 2026.5.27`（覆盖 GHSA-8wg3-5mcm-fjq8 / GHSA-83w9-h5wv-j9xm） |
| 磁盘 | 至少 10 GB（向量库 + AGFS 文件） |
| 内存 | 推荐 8 GB+（embedding 服务是常驻内存的） |
| 外部账户 | 火山方舟 Ark（推荐）/ OpenAI / Ollama / Kimi / GLM，至少开通 VLM + 多模态 Embedding |
| 网络 | 本机 `127.0.0.1:1933`（OpenViking）和 `127.0.0.1:18789`（OpenClaw gateway）必须可访问 |

> **不要在生产环境把 OpenViking 绑 `0.0.0.0` + 弱 key**。dev 用 `127.0.0.1` 即可，OpenClaw 也在本机 → 走 loopback。生产请配 `root_api_key` + 反向代理。

---

## 3. 选型：用什么 VLM 和 Embedding

OpenViking 的 `ov.conf` 允许你**自由替换** VLM 和 Embedding provider，常见组合：

| 场景 | VLM | Embedding | 备注 |
| --- | --- | --- | --- |
| **国内生产 (推荐)** | 火山方舟 `doubao-seed-2-0-lite-260215` | 火山方舟 `doubao-embedding-vision-251215` (2048 dim) | 走 `https://ark.cn-beijing.volces.com/api/v3`；多模态真多模态（不是 mock） |
| 国外生产 | OpenAI `gpt-4o-mini` | OpenAI `text-embedding-3-large` | 注意 OpenAI embedding 纯文本，没有真多模态向量 |
| 全本地零成本 | Ollama `llava` | Ollama `nomic-embed-text` (768 dim) | 走 `http://localhost:11434/v1`；注意 `input: text` 不会真多模态 |
| 编程订阅 | Codex / Kimi / GLM Coding | 上面任一 | 走订阅 token（`openviking-server init` 会引导登录） |

**实测可用的 VLM 候选**（火山方舟 `chat/completions` 上 2026-07 验证）：

| Model | 状态 |
| --- | --- |
| `doubao-seed-2-0-lite-260215` | ✅ 可用 |
| `doubao-seed-2-0-lite-260428` | ⚠️ example 默认值，实测有时返回 404 |
| `doubao-seed-evolving` | ✅ 可用（走 `/v3/responses` 端点，Thinking mode） |

> **建议**：用 `doubao-seed-2-0-lite-260215` 作 VLM，`doubao-embedding-vision-251215` 作 embedding。本指南后续示例都用这套。

---

## 4. 部署 OpenViking 服务（多模态版）

### 4.1 安装 + 初始化

```bash
pip install openviking --upgrade --force-reinstall
openviking-server init         # 生成 ~/.openviking/ov.conf
openviking-server doctor       # 检查本地依赖（python / ffmpeg / pillow …）
```

`init` 写出的 `ov.conf` 是**纯文本版**（不是 example 里的 JSON）。如果你想从 `examples/ov.conf.example` 改，**只改需要的字段**，不要全文替换——`init` 写出的配置含本机路径和 token。

### 4.2 编辑 `ov.conf`（最少改动版）

最小可用多模态配置（**红色**字段是必须改的）：

```json
{
  "server": {
    "host": "127.0.0.1",           // ⚠️ dev 用 127.0.0.1，别用 0.0.0.0
    "port": 1933,
    "root_api_key": null           // 多人共用再设；单人本机用 API key 模式
  },
  "embedding": {
    "dense": {
      "model": "doubao-embedding-vision-251215",     // 红色：选的多模态 embedding
      "api_key": "填你的 Ark API key",                // 红色：API key
      "api_base": "https://ark.cn-beijing.volces.com/api/v3",
      "dimension": 2048,                              // ⚠️ doubao-embedding-vision 是 2048，example 写 1024 是错的
      "provider": "volcengine",
      "input": "multimodal"                           // 关键：multimodal 才会真多模态向量
    },
    "text_source": "summary_only",                    // 文本文件只 embed 摘要，省 token
    "image_vectorization": "summary_only"             // 图默认只 embed 摘要；想真多模态改 "image_and_summary"
  },
  "vlm": {
    "model": "doubao-seed-2-0-lite-260215",           // 红色：选的 VLM
    "api_key": "填你的 Ark API key",                   // 红色：API key（与 embedding 可以同一个）
    "api_base": "https://ark.cn-beijing.volces.com/api/v3",
    "temperature": 0.0,
    "max_retries": 2,
    "provider": "volcengine",
    "thinking": false
  },
  "parsers": {
    "image": {
      "enable_ocr": false,                            // 默认不跑 OCR（中文 OCR 用 volcengine OCR 服务要单独开）
      "enable_vlm": true,                             // ✅ 必须开
      "ocr_lang": "eng",
      "vlm_model": "doubao-seed-2-0-lite-260215",     // 红色：VLM 模型名（与上面 vlm.model 一致）
      "max_dimension": 2048
    },
    "audio": {
      "enable_transcription": true,
      "transcription_model": "whisper-large-v3",     // ⚠️ 默认值；要真跑通需要装 faster-whisper / volcengine ASR
      "language": null,
      "extract_metadata": true
    },
    "video": {
      "extract_frames": true,
      "frame_interval": 10.0,
      "enable_transcription": true,
      "enable_vlm_description": true,                // ⚠️ 默认 false，要显式开才有视频理解
      "max_duration": 3600.0
    },
    "pdf": {
      "strategy": "auto",
      "maxeru_endpoint": "https://mineru.example.com/api/v1",   // 红色：换成你的 mineru 部署
      "maxeru_api_key": "填 mineru key",                          // 红色
      "maxeru_timeout": 300.0
    }
  },
  "default_search_mode": "quick",     // ⚠️ "thinking" 会调 rerank，没配 rerank 会失败
  "log": { "level": "INFO" }
}
```

**绝对会踩的坑**：

1. **`embedding.dense.dimension` 写错**。`doubao-embedding-vision-251215` 的真实维度是 **2048**，example 默认 `1024` 是错的。错一次，所有已有向量要重算。
2. **`vlm.model` 用 example 默认的 `doubao-seed-2-0-lite-260428` 实测 404**。换 `doubao-seed-2-0-lite-260215` 或 `doubao-seed-evolving`。
3. **`parsers.video.enable_vlm_description` 默认 `false`**。你以为开了多模态，其实视频帧只抽不描述。
4. **`parsers.pdf.mineru_endpoint` 不改就用不了 PDF**。要么自己部署 mineru，要么把 PDF 显式转 markdown 再 add_resource。
5. **`default_search_mode: "thinking"` 会强行调 rerank**。没配 `rerank` 段时改 `"quick"`，不然 `/v3/search` 直接 500。

### 4.3 启动 + 自检

```bash
# 前台（看日志）
openviking-server

# 后台
mkdir -p ~/.openviking/data/log
nohup openviking-server > ~/.openviking/data/log/openviking.log 2>&1 &

# 健康检查
curl http://127.0.0.1:1933/health
# 期望: {"status":"ok", ...}

# 验证 VLM key（用 OpenViking 自带 client）
python -c "import openviking as ov; c = ov.OpenViking(path='./data'); c.initialize(); print(c.health())"
```

**多模态冒烟测试**（最快验证 VLM + embedding 链路）：

```python
# smoke_multimodal.py
import openviking as ov
client = ov.OpenViking(path="./data")
client.initialize()

# 1. 喂一张图
res = client.add_resource(path="./test.jpeg", wait=True)
print("root_uri:", res["root_uri"])

# 2. 看 .abstract.md 是否生成（说明 VLM 调通了）
import openviking_cli
files = client.ls(res["root_uri"])
for f in files:
    if not f.get("isDir"):
        print(f["name"])

# 3. 文字搜图（说明 embedding 调通了）
hits = client.find("图片里描述的对象", target_uri="viking://resources", limit=3)
for h in hits.resources:
    print(h.uri, h.score)
```

---

## 5. 部署 OpenClaw + 打开多模态工具

### 5.1 装包

```bash
npm install -g openclaw@2026.5.27     # 或更新
openclaw --version

# ⚠️ 装的是 plugin 不是 skill
openclaw plugins install clawhub:@openviking/openclaw-plugin
# 不要用: clawhub install openviking   ← 装的是 AgentSkill 不是 plugin
```

### 5.2 配置（关键一步：开 `add_resource`）

```bash
# 1. 写 baseUrl + apiKey
openclaw openviking setup \
    --base-url http://127.0.0.1:1933 \
    --api-key "$OPENVIKING_API_KEY" \
    --json

# 2. ⚠️ 关键：把 add_resource 工具暴露给 agent
openclaw config set plugins.entries.openviking.config.enableAddResourceTool true
```

**`enableAddResourceTool: true` 是打开多模态的"开关"**。默认 `false` ——
不打开这个，agent 就只能读已经存在的资源，不能 import 新文件。`/add-resource` slash 命令仍然能用，但 agent 自动调用时会被拒。

其它建议一并打开的：

```bash
# 让 recall 能搜到 viking://resources (agent 共享知识库)
openclaw config set plugins.entries.openviking.config.recallResources true
openclaw config set plugins.entries.openviking.config.recallTargetTypes "user,agent,resource"
```

### 5.3 重启 gateway + 验证

```bash
openclaw gateway restart
openclaw openviking status --json
```

期望输出（关键字段）：

```json
{
  "configured": true,
  "slotActive": true,
  "health": { "ok": true, ... }
}
```

### 5.4 端到端验证多模态

**手动 slash 命令**（先验证 CLI 走得通）：

```bash
# 上传图
/add-resource /path/to/photo.jpg
# 上传文档
/add-resource /path/to/spec.pdf
```

**通过 agent 验证**（让 agent 自己调 `add_resource`）：

```
用户: 把 D:\samples\view.jpeg 加进记忆库，然后告诉我这张图里有什么

agent 行为: 
  1. 自动调 add_resource 工具（仅 enableAddResourceTool=true 时才允许）
  2. 等待 OpenViking 跑完 VLM + embedding
  3. 调 ov_search 或 memory_recall 反查
  4. 把 .abstract.md 内容回答给用户
```

**没有这一步之前，不要宣布"多模态已打开"**。

---

## 6. 三种 `image_vectorization` 模式

控制"图到底怎么向量化"：

| 模式 | 向量内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| `summary_only`（默认） | 只把 VLM 生成的 .abstract.md 文本 embed 成向量 | 便宜，跨模态检索够用 | 不是"真多模态向量"，是文本向量 |
| `image_only` | 直接用多模态 embedding 把图像 encode 成向量 | 真多模态检索（"以图搜图"） | 必须 provider 支持 `input: multimodal`（如 `doubao-embedding-vision-251215`） |
| `image_and_summary` | 同时把图像和摘要文本 embed，融合 | 召回最准 | 成本翻倍 |

**推荐**：

- 通用场景 → 保持 `summary_only`，配合 `image_and_summary` 在召回阶段再 fine-tune
- 真要"以图搜图" → 改 `image_and_summary`
- 资源紧张 / Provider 不支持 multimodal embedding → 保持 `summary_only`

切换方法：改 `ov.conf` 里 `embedding.image_vectorization`，**然后重建向量库**（改 mode 不会自动重 embed 历史数据）。

---

## 7. 故障排查速查

| 症状 | 原因 | 解法 |
| --- | --- | --- |
| `add_resource` 上传图成功，但召回是 0 分 | VLM 调通但 embedding 失败 / dimension 不匹配 | 看 `~/.openviking/data/log/openviking.log` 里的 `embedding` 错误；确认 `dimension: 2048` |
| `.abstract.md` 没生成 | `parsers.image.enable_vlm: false` 或 VLM key 错 | 显式开 `enable_vlm: true`；用 `openviking-server doctor` 验证 VLM |
| agent 调 `add_resource` 报 "tool not allowed" | `enableAddResourceTool` 没开 | `openclaw config set plugins.entries.openviking.config.enableAddResourceTool true` + `gateway restart` |
| 视频 recall 不到 | `parsers.video.enable_vlm_description: false` | 显式开 true；需要 ffmpeg 在 PATH |
| PDF 解析报 `mineru_endpoint` 错 | 默认 endpoint 是 example 占位 | 自己部署 mineru 或改用 `mineru` 之外的 PDF 工具（先转 markdown 再 add_resource） |
| `/v3/search` 报 500 | `default_search_mode: "thinking"` 但没配 rerank | 改 `"quick"`，或配 `rerank.*` 段 |
| OpenViking 服务起不来，端口被占 | 旧进程没死 | `lsof -i:1933` / `Get-Process -Id (Get-NetTCPConnection -LocalPort 1933).OwningProcess` 然后 kill |
| 多模态 embedding 返回 401 | Ark key 没开 multimodal 权限 | 火山方舟控制台 → 在线推理 → 开通 `doubao-embedding-vision-251215` 模型的"多模态输入"权限 |
| OpenClaw gateway 启动后立刻退出 | OpenClaw 版本 < 2026.5.27 | 升级到 `>= 2026.5.27`（GHSA-8wg3-5mcm-fjq8 修复线） |

---

## 8. 一键验收脚本

部署完跑一下这个，把 4 节 + 5 节全跑通：

```python
# verify_multimodal.py
import openviking as ov
import subprocess, json

# 1. OpenClaw plugin 在线
status = json.loads(subprocess.check_output(
    ["openclaw", "openviking", "status", "--json"], text=True
))
assert status["slotActive"], "OpenClaw slot 没激活"
assert status["health"]["ok"], f"OpenViking 服务不健康: {status}"
print("✅ OpenClaw slot active + OpenViking healthy")

# 2. 多模态 add_resource
client = ov.OpenViking(path="./data")
client.initialize()
res = client.add_resource(path="./test.jpeg", wait=True)
root = res["root_uri"]
print(f"✅ add_resource 成功: {root}")

# 3. .abstract.md 生成
files = [f["name"] for f in client.ls(root) if not f.get("isDir")]
assert any(".abstract.md" in f for f in files), f"VLM 没跑: {files}"
print("✅ VLM 描述生成 (.abstract.md)")

# 4. 跨模态检索
hits = client.find("你期望的描述", target_uri="viking://resources", limit=3)
assert any(root in h.uri for h in hits.resources), f"没召回刚上传的图: {[h.uri for h in hits.resources]}"
print(f"✅ 跨模态检索命中: {hits.resources[0].uri} (score {hits.resources[0].score:.3f})")

print()
print("🎉 多模态端到端验收通过")
```

**期望输出**：

```
✅ OpenClaw slot active + OpenViking healthy
✅ add_resource 成功: viking://resources/test_jpeg
✅ VLM 描述生成 (.abstract.md)
✅ 跨模态检索命中: viking://resources/test_jpeg/.abstract.md (score 0.812)
🎉 多模态端到端验收通过
```

任意一行失败 = 还没真"打开多模态"，回去看 `~/.openviking/data/log/openviking.log` + `~/.openclaw/openviking/recall-traces/`。

---

## 9. 配置速查表

### OpenViking `ov.conf` 多模态相关字段

| 段 | 字段 | 推荐值 | 说明 |
| --- | --- | --- | --- |
| `embedding.dense` | `model` | `doubao-embedding-vision-251215` | 多模态 embedding 模型 |
| `embedding.dense` | `dimension` | `2048` | ⚠️ 不要照抄 example 的 1024 |
| `embedding.dense` | `input` | `multimodal` | 关键；切到多模态向量 |
| `embedding` | `image_vectorization` | `summary_only` / `image_only` / `image_and_summary` | 图怎么向量化 |
| `embedding` | `text_source` | `summary_only` | 文本文件 embed 摘要 |
| `vlm` | `model` | `doubao-seed-2-0-lite-260215` | VLM 模型 |
| `parsers.image` | `enable_vlm` | `true` | 跑视觉理解 |
| `parsers.image` | `vlm_model` | 同 `vlm.model` | image parser 用的 VLM |
| `parsers.image` | `enable_ocr` | `false` | 默认 false；中文 OCR 需另接 |
| `parsers.audio` | `enable_transcription` | `true` | 跑 ASR |
| `parsers.audio` | `transcription_model` | `whisper-large-v3` | 需装 faster-whisper |
| `parsers.video` | `enable_vlm_description` | `true` | ⚠️ 默认 false |
| `parsers.video` | `enable_transcription` | `true` | 视频音轨转写 |
| `parsers.pdf` | `mineru_endpoint` | 你的 mineru URL | 必填 |
| `parsers.pdf` | `mineru_api_key` | 你的 mineru key | 必填 |
| - | `default_search_mode` | `quick` | 避开 rerank 依赖 |

### OpenClaw plugin 关键字段

| 字段 | 默认 | 多模态推荐 | 说明 |
| --- | --- | --- | --- |
| `enableAddResourceTool` | `false` | **`true`** | **没这个 agent 不能自动 add_resource** |
| `recallResources` | `false` | `true` | 召回时搜 `viking://resources` |
| `recallTargetTypes` | `[user, agent]` | `[user, agent, resource]` | 把资源纳入召回 |
| `autoCapture` | `false` | `true` | 自动从对话里 capture 记忆 |
| `autoRecall` | `false` | `true` | 召回时自动注入上下文 |

修改方法：

```bash
openclaw config set plugins.entries.openviking.config.<field> <value>
openclaw gateway restart
openclaw openviking status --json
```

---

## 10. 后续可选优化

部署完跑通"端到端验证"后，可以再考虑：

- **生产 hardening**：配 `server.root_api_key` + 反代；OpenClaw `peer_role: person` 区分多用户
- **细粒度召回**：调 `recallLimit` / `recallScoreThreshold` / `recallMaxInjectedChars`
- **持久化 trace**：`traceRecall: true` + `traceRecallPersist: true` 排查召回问题
- **多租户**：`recallTargetTypes: ["user", "agent"]` 改成 `["resource"]` 切换到共享知识库模式
- **Recall trace 分析**：参考 [`recall-trace-api.md`](./recall-trace-api.md)

---

## 相关文档

- [`INSTALL.md`](../INSTALL.md) — OpenClaw 插件安装主流程
- [`INSTALL-ZH.md`](../INSTALL-ZH.md) — 上述中文版
- [`openviking-plugin-reference.md`](./openviking-plugin-reference.md) — OpenClaw 插件配置全字段参考
- [`../ov.conf.example`](../ov.conf.example) — OpenViking 配置文件模板
- [`openviking-runtime-query-config.md`](./openviking-runtime-query-config.md) — 运行时查询配置
