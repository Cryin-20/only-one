# FACT.md - Cherry AI 持久知识库

> 最后更新：2026-09-12（v13 — 加 only-one v0.3.16 "回复说到一半停"根因与修复 + agent loop 关键参数表）
> 维护规则：6+ 个月重要的写入这里，一次性的写 JOURNAL.jsonl
> **边界**：本文件只装事实/决策/技术参数；AI 行为规则 → 进 Cherry Studio system prompt，不在本文件
> **凭据边界**：本文件 / JOURNAL / SOUL / USER 🚫 严禁明文 API key / 密码 / SSH 私钥内容。凭据路径索引见 `memory/secrets-index.md`

---

## 🆕 only-one v0.3.16 修复"回复说到一半停"（2026-09-12）

- **首录**：2026-09-12
- **症状**：云端 agent 长回复说到一半就停（短回复正常）；Cherry 工作台正常
- **commit**：`c891e18 v0.3.16-agent-loop-finalize`

### 真根因：`MAX_AGENT_ROUNDS` 默认 5 + 无收尾机制

模型倾向**多轮工具调用**（查环境 / 文件 / 网络）。`for round_idx in range(max_rounds)` 跑满后循环自然结束 → **模型永远没机会输出最终答案** → 页面上只留一句开场白 + 一堆工具卡片。
用户原话"总是在 **5 或者是 6** 就停止"—— **精确对应 `max_rounds=5`**（第 6 次机会被剥夺）。

### 修复内容（4 项 P0）

1. **P0-1 finalize pass**：循环跑满且仍有 pending tool_calls → 追加一次**不带 tools** 的请求，prompt 强制"基于全部工具结果直接输出文字答案"，杜绝无答案
2. **P0-2** `MAX_AGENT_ROUNDS` 5 → **25**（写进 `.env` 顶部安全区）
3. **P0-3** 捕获 `finish_reason`，为 `length` 时发 `truncated` 事件（截断可见）
4. **P0-4** `generate()` 补真正的 `except Exception`（见下方"假修复"教训）

### 验证（同一问题，3 组对照）

| 配置 | 结果 |
|---|---|
| 旧（rounds=5，无收尾）| **26 字符，无答案** ❌ |
| 新（rounds=25）| 957 字符完整答案 ✅ |
| 新（强制 rounds=2）| `finalize_start` 触发 → 849 字符答案 ✅ |

### 历史根因（另一条，已修）

09-11 有 **14 条** assistant 消息 `completion_tokens=2048` 且**全部截断** ← `.env` 变量写在"空行 + `#` 注释"之后 → systemd `EnvironmentFile` 停止解析 → 代码 `os.environ.get("LLM_MAX_TOKENS", "2048")` **落回默认 2048**。v0.3.14 把变量移到 `.env` 顶部后修复。

### only-one agent loop 关键参数（复用必读）

| 参数 | 位置 | 当前值 | 说明 |
|---|---|---|---|
| `MAX_AGENT_ROUNDS` | `/home/ubuntu/only-one/.env` | **25** | 工具轮次上限；超限触发 finalize pass |
| `LLM_MAX_TOKENS` | `.env` **顶部** | **131072** | 单次响应输出上限；**必须在 .env 顶部**（systemd 解析坑）|
| `httpx` timeout | 代码 `call_llm_stream` | 60s | 流式读超时（chunk 间隔）|

### 已知未修（P1，待拍板）

- **`tool_call_start` 事件洪水**：`arguments` **每字符**发一个 SSE 事件（单轮可达 400+），前端压力大
- **前端 round 0 归位**：单轮回复内容全进灰色斜体"思考区"，"回答区"永远空白

---

## 🆕 only-one 接入 DeepSeek provider（2026-09-12）

- **首录**：2026-09-12
- **目标**：云端 only-one（`152.136.41.22:8099`）新增 DeepSeek 模型
- **拍板**：A1（key 走本地文件 `D:\secrets\deepseek.env`）+ B1（加 only-one）+ C1（`deepseek-flash`）+ D1（不切默认）

### 落地结果

| 项 | 值 |
|---|---|
| `.env` | `/home/ubuntu/only-one/.env` **第 1 行** `DEEPSEEK_API_KEY`（插第 1 行是为了避开 systemd EnvironmentFile "注释/空行后停止解析"的旧坑）|
| DB | `data/only-one.db` → `provider_models.deepseek`：base_url `https://api.deepseek.com`、model `deepseek-flash`、context_limit `1048576`、is_active `0` |
| 加密 | `api_key_enc` 走 `PROVIDER_KEY_MASTER_SECRET` + Fernet（enclen=140）|
| 默认模型 | 仍是 **minimax**（D1，未切）|
| 备份 | `/home/ubuntu/only-one/.env.bak-20260912-deepseek` |
| 安全 | `.env` 被 `.gitignore` 的 `*.env` 覆盖且不在 git index ✅ |

### DeepSeek API 现状（2026-09-12 抓官方文档 + 实测）

- 模型名：**`deepseek-flash`**（V4.1-Flash）/ **`deepseek-v4-pro`**；旧名 `deepseek-chat` 已从文档消失
- base_url（OpenAI 格式）：**`https://api.deepseek.com`**（**不带 `/v1`**）；Anthropic 格式：`https://api.deepseek.com/anthropic`
- 上下文 1M / 输出最大 384K / 两个模型都支持 Tool Calls、Anthropic API、Responses API
- 实测：`GET https://api.deepseek.com/models` → HTTP 200
- **思考模式默认开启**：`reasoning_content` 与 `content` 是不同字段；only-one 目前**只取 `content`**（reasoning 被丢弃）

### only-one 多 provider 架构（复用必读）

- `app/provider_store.py`：SQLite 表 `provider_models` 持久化模型注册表，key 用 Fernet 加密（master secret 来自 `PROVIDER_KEY_MASTER_SECRET`）
- key 解析优先级：请求体 `provider_key`（浏览器 session，不落库）> DB `api_key_enc` > 环境变量（**`_env_key()` 只读 `os.environ`，没有 .env 文件 fallback**，与 `_master_secret()` 不同）
- API 端点：`GET /api/models`、`GET /api/llm/providers`、`POST /api/models`、`PATCH /api/models/{model_key}`、`POST /api/models/{model_key}/activate`、`DELETE /api/models/{model_key}`
- **⚠️ 复用坑**：调用 `provider_store.resolve()` / `public_models()` / `configured_keys()` 前必须先 `conn.row_factory = sqlite3.Row`，否则 `TypeError: tuple indices must be integers or slices, not str`
- **⚠️ 遗留**：`BUILTIN_DEFAULTS` 里 deepseek 仍是旧值（`deepseek-chat` + `65536` + `https://api.deepseek.com/v1`），本次只改了 DB 行、**源码未改**（只影响未来新部署）
- 服务：systemd `only-one.service`，`EnvironmentFile=/home/ubuntu/only-one/.env`，`ExecStart=venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8099`

### 云端仓库与 GitHub 的关系（2026-09-12 查证）

| 路径 | remote |
|---|---|
| `/home/ubuntu/only-one/`（主仓库，跑服务）| **无 remote** ❌ |
| `/home/ubuntu/only-one/knowledge-base/`（子目录）| `https://github.com/Cryin-20/only-one.git` ✓ |

→ 云端主仓库**没有** GitHub 同步通道；GitHub 上 `Cryin-20/only-one` 存在（public，分支 `main` + `cherry-memory`）。

---

## 🆕 v2 only-one 完整复盘（2026-09-10）

- **首录**：2026-09-10
- **末更**：2026-09-10
- **来源**：v2 only-one v0.3.0-batch-upgrade 完工后的复盘
- **目的**：给未来 v3 / 类似项目参考

### Skill 文件位置

`C:\Users\Cryin\AppData\Roaming\CherryStudio\Data\Agents\6c6b9e7b-a53a-40ec-919a-84b4f4f34293\memory\skills\v2-only-one-retrospective.md`

### 5 个核心结论

1. **先问"跑顺序"再开工**（3 步以上任务必须追问顺序）
2. **AI 主动建议 + 用户拍板**（不是 AI 替用户决定）
3. **端到端测试才能发现真问题**（C 验证发现 kb-pull.sh v1 的 2 个真问题）
4. **secure-token-handling Skill 必须独立**（6 次教训 → 自动加载 → 套模板）
5. **失败是资产不是负债**（10 次失败沉淀 → 未来 v3 不再踩）

### 17 条关键决策 + 10 条教训（详见 Skill 文件）

### 自我评分：82%（达成 Cherry 80% 目标）

### 时间 / 资源统计

- 12 小时 / 3 天跨度
- 4 个 commit
- ~150KB 代码
- 9 个 tools
- 4 个 Skill 文件
- ~15 条 JOURNAL
- **10 次失败**（沉淀成资产）

### 关联

- SOUL 观点 F + G（Skill 机制）
- SOUL 观点 H（安全底线 v2 修订 C1）
- v0.3.0-batch-upgrade tag
- 4 个 Skill 文件

---

## 🆕 secure-token-handling Skill 抽取 v2（2026-09-10）

- **首录**：2026-09-10
- **末更**：2026-09-10
- **来源**：6 次 token 教训合并
- **文件**：`memory/skills/secure-token-handling.md`

### 6 次历史教训

| # | 时间 | 事件 |
|---|---|---|
| 1 | 2026-09-08 | Doubao Seedream key 进对话 |
| 2 | 2026-09-10 | PowerShell Read-Host 显示密码 |
| 3 | 2026-09-10 | GitHub PAT 明文进 console |
| 4 | 2026-09-10 | read_text_file 输出 token 明文 |
| 5 | 2026-09-10 | kb-pull.sh 把 token 存到 .git/config |
| 6 | 2026-09-10 | git remote -v 输出 token 明文 |
| 7 | 2026-09-12 | `cat webchat.pm2.json` 输出 MiniMax key 明文（**待用户轮换该 key**）|

### 关键规则

| 行为 | 是否允许 |
|---|---|
| AI 读 .env + 输出长度/前 8/后 4 | ✅ |
| AI 用 SCP 文件模式传 token | ✅ |
| AI 用 SecureString 输入 | ✅ |
| **AI 输出 token 明文** | ❌ 核心防御 |
| AI 用 read_text_file 读 .env | ❌ |
| **cat / od / hexdump 含 key 的文件** | ❌ 必须先脱敏或用 `--names-only` |

---

## 🆕 kb-pull.sh v2 优化（2026-09-10）

- 4 个改进：GIT_ASKPASS / retry / fallback / audit metric
- 验证：PULLED (attempt 1): HEAD=35045c2

---

## 🆕 v2 only-one v0.3.0-batch-upgrade tag（2026-09-10）

- commit `307ba41` + tag `v0.3.0-batch-upgrade`
- 8 files / ~95KB backend + ~37KB frontend
- 达成 **80% Cherry** 里程碑

---

## 🆕 信息冗余削减规则（2026-09-10，配套 SOUL 观点 J）

- **目的**：减少"每次新对话重复问 key 在哪 / 云端在哪 / 域名是啥"
- **索引文件**：`memory/secrets-index.md`（已扩展为"高频信息索引"，凭据类 + 非凭据类）
- **触发阈值**：同一类信息**出现 2 次** → AI 主动问"以后常用吗？要收录吗？"
- **主动建议**：用户**第 1 次**提到明显高频可能信息（域名 / ssh 主机 / 端口 / 服务地址）→ AI 也主动问
- **收录由用户拍板**（AI 不偷偷收）
- **下次使用**：AI 主动查 `secrets-index.md`，不再问用户

### 4 字段规范（所有条目统一）

| 字段 | 含义 |
|---|---|
| 名称 | 条目简称 |
| 类别 | ssh 主机 / 域名 / API key / 端口 / 项目路径… |
| 值 / 路径 | 凭据类只写路径；非凭据类可写明文 |
| 变量名 | shell / .env 变量名（可选） |
| 使用场景 | 什么时候用到 |
| 备注 | 注意事项 / 关联文件 |

### 关联

- SOUL 观点 J（行为规则）
- `memory/secrets-index.md`（索引本身）
- 沉淀走观点 G 决策表

---
