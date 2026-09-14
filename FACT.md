# FACT.md - Cherry AI 持久知识库

> 最后更新：2026-09-14（v15 — 群聊模式补：页面视觉与主页面统一、@B 路由修正、主页入口）
> 上一版：v14 加 only-one 群聊房间模式 v0.4.0-room（架构 / 参数 / API / 运维要点）
> 维护规则：6+ 个月重要的写入这里，一次性的写 JOURNAL.jsonl
> **边界**：本文件只装事实/决策/技术参数；AI 行为规则 → 进 Cherry Studio system prompt，不在本文件
> **凭据边界**：本文件 / JOURNAL / SOUL / USER 🚫 严禁明文 API key / 密码 / SSH 私钥内容。凭据路径索引见 `memory/secrets-index.md`

---

## 🆕 only-one 群聊房间模式 v0.4.0-room（2026-09-14）

- **首录**：2026-09-14
- **目标**：在 only-one 上做「群聊式多 agent」—— A 执行 / B 代理人类回复 / 人类可插话
- **状态**：已上线，端到端验收通过；用户已实际使用

### 架构

| 层 | 实现 |
|---|---|
| 房间 | `sessions` 表加 `kind='room'`（现有单会话页零影响） |
| 消息 | `messages` 表加 `speaker`（`user`/`A`/`B`）；`role` 字段不动 |
| 游标 | `sessions.room_cursor`（已消费到哪条）+ `room_pending`（谁该接着说），均持久化 |
| 调度器 | `app/rooms.py::scheduler_loop()`，进程内 asyncio 常驻，1 秒轮询 |
| 长连接 | `GET /api/rooms/{id}/stream`（SSE；前端 fetch+ReadableStream 消费，2.5s 自动重连） |
| 前端 | `static/room.html`，入口 `/room`（主页侧边栏 + 顶栏两处入口） |
| 挂载 | `main.py` 仅改 3 处：startup 挂调度器 + shutdown 钩子 + 文件末尾 `rooms.register(app)` |

### 角色卡（rooms.py 顶部常量）

- **A**：执行 Agent，带工具（复用 `main.TOOLS_SCHEMA` + `main.execute_tool`），决策点呼叫 B，完成时输出 `[DONE]`
- **B**：代理 Agent，不带工具，代表人类回 A；回复须含 结论 / 理由 / 置信度 / 风险；高风险或低置信 `@HUMAN` 升级

### 关键参数（rooms.py 顶部）

| 参数 | 值 | 说明 |
|---|---|---|
| `POLL_INTERVAL` | 1.0s | 调度器轮询间隔 |
| `MAX_AUTO_ROUNDS` | 12 | 同房间自动往返上限，超限暂停等人 |
| `MAX_TOOL_ROUNDS` | 6 | 单 turn 内工具轮数上限 |
| `TOKEN_BUDGET` | 24000 | 单次上下文 token 预算（从最新往回取） |
| `TOOL_RESULT_CHARS` | 4000 | 工具结果进上下文的截断长度 |
| `REPEAT_GUARD` | 2 | 相同问题哈希重复次数 → 暂停 |

### 路由规则（`_route`）

- **人类发言**：含 `@B`（且无**行首** `@A`）→ B；否则 → A
- **A 发言**：含 `[DONE]` → 停；**行首** `@HUMAN` → 停（等人类）；`@B` 或决策关键词 → B；否则停
- **B 发言**：**行首** `@A` → A（B 在给 A 下指示）；否则停（B 在直接回答人类）
- ⚠️ 判定 @ 提及一律区分「行首」与「行内」：行首才算真的在呼叫，行内的往往是 agent 在解释规则

### 状态机（`sessions.room_status`）

`running` / `waiting_human`（升级人类）/ `paused`（轮次·重复·超时）/ `done`（`[DONE]`）
人类发消息自动解除 `waiting_human` / `done` → `running`；`paused` 需显式 `/resume`
⚠️ `[DONE]` 判定**优先于** `@HUMAN`（两者同时出现以任务完成为准）

### API

```
POST   /api/rooms                 建房 {title} -> {room_id}
GET    /api/rooms                 房间列表
GET    /api/rooms/{id}/messages   历史（含 speaker / tool_calls / token）
POST   /api/rooms/{id}/messages   人类发言（排队语义，不打断当前 turn）
GET    /api/rooms/{id}/stream     SSE 长连接
POST   /api/rooms/{id}/pause | /resume | /interrupt
GET    /api/rooms/{id}/state      状态（cursor / pending / rounds / tokens / busy）
```

### 前端（room.html）与主页面的关系

- **视觉与 index.html 同源**：完整复制了 `:root/[data-theme="minimal"]` 与 `[data-theme="cyber"]` 变量块，
  以及 `.header` / `.header-btn` / `.chat` / `.message` / `.bubble` / `.input` / `.empty` 等组件样式
- **主题共享**：读写同一个 localStorage 键 `only-one-theme`，在任一侧切主题，另一侧跟随
- **消息结构复用**：`.message.user` / `.message.A` / `.message.B` + `.bubble`（主页面只有 user/ai 两态）
- cyber 主题下 A/B 用 `::before` 前缀（`A >>>` / `B >>>`）区分，`minimal` 下用 `.who` 标签
- 入口：主页侧边栏 `.room-entry` 卡片 + 顶栏 `a.room-btn` 按钮；群聊页左上角「← 单会话」返回

### 运维要点

- **systemd drop-in（必须）**：`/etc/systemd/system/only-one.service.d/graceful.conf`
  → `--timeout-graceful-shutdown 5` + `TimeoutStopSec=20`
  （不加则有活跃 SSE 连接时 `systemctl restart` 卡满 90 秒才被 SIGKILL）
- 脚本：`scripts/room_smoke.py`（冒烟）、`scripts/room_diag.py`（诊断房间/消息/工具调用）
- 备份：`/home/ubuntu/only-one-backup-20260914-001201.tar.gz` + DB `data/only-one.db.bak-pre-room-20260914-001203`
- 回滚点：`app/main.py.bak-20260914-pre-room`、`app/main.py.bak-pre-shutdown`、
  `static/index.html.bak-20260914-pre-roomlink`、`static/room.html.bak-20260914-pre-backlink`
- 房间即 session：删除走现有 `DELETE /api/sessions/{id}`（前端暂无删除入口）

### 实测数据

- A/B 自动轮转 3 轮 ≈ 3.3K tokens（含工具调用）；B 单轮回答 ≈ 400 tokens
- 加裁剪前单 turn 曾冲到 24K tokens（A 一轮连调 13 次工具）；加 `TOOL_RESULT_CHARS` + 提示词节制后稳定在 3–7K

### 已知限制（第一版）

- 上下文裁剪 = 按 token 预算从最新往回取，**无摘要压缩**
- 无决策卡片 / 风控分级界面（按计划留第二版）
- 轮次计数在内存，重启后归零
- 房间页与单会话页同样**无登录认证**（知道 IP 即可访问）

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
| DB | `data/only-one.db` → `provider_models.deepseek`：base_url `https://api.deepseek.com`、model `deepseek-flash`、context_limit `1048576`、is_active `0`（2026-09-14 复查：**已切为 active**，`/api/models` 返回 `"active":"deepseek"`）|
| 加密 | `api_key_enc` 走 `PROVIDER_KEY_MASTER_SECRET` + Fernet（enclen=140）|
| 默认模型 | **deepseek**（2026-09-14 实测已激活；minimax 转为 inactive）|
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
- **⚠️ 遗留**：`BUILTIN_DEFAULTS` 里 deepseek 仍是旧值（`deepseek-chat` + `65536` + `https://api.deepseek.com/v1`），源码未改（只影响未来新部署）
- 服务：systemd `only-one.service`，`EnvironmentFile=/home/ubuntu/only-one/.env`，`ExecStart=venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8099`（+ drop-in 的 `--timeout-graceful-shutdown 5`）

### 云端仓库与 GitHub 的关系（2026-09-12 查证）

| 路径 | remote |
|---|---|
| `/home/ubuntu/only-one/`（主仓库，跑服务）| **无 remote** ❌ |
| `/home/ubuntu/only-one/knowledge-base/`（子目录）| `https://github.com/Cryin-20/only-one.git` ✓ |

→ 云端主仓库**没有** GitHub 同步通道；GitHub 上 `Cryin-20/only-one` 存在（public，分支 `main` + `cherry-memory`）。

### only-one 代码结构速查（2026-09-14 实测）

| 文件 | 规模 | 说明 |
|---|---|---|
| `app/main.py` | 83KB / ~1820 行 | 单体：DB 初始化、`/api/chat`（agent loop + SSE）、工具执行、会话/目标/技能/文件 API |
| `app/prompts.py` | 15KB | `SYSTEM_PROMPT` + `SKILL_AUTO_TRIGGER_RULE` + `GOAL_SYSTEM_RULE` + `DUAL_WORKFLOW_RULE`（双轨：手动档/自动档）|
| `app/provider_store.py` | 9.7KB | 多 provider 注册表 + Fernet 加密 |
| `app/phase4_tools.py` | 8.3KB | `estimate_tokens` / `inspect_file` / `create_office_file` / `crawl_public_site` |
| `app/skills.py` | 4.2KB | skill 列表 / 加载 / 匹配 |
| `app/rooms.py` | 32KB | **群聊房间模式（2026-09-14 新增）** |
| `static/index.html` | 81KB | 单会话页（`addMessage(role)` 只认 user/ai；流式用 `fetch + response.body.getReader()`，非 EventSource）|
| `static/room.html` | 21KB | **群聊页（新增，与 index.html 同一套主题变量）** |
| `app/routes/` | 空目录 | 所有路由都在 main.py |

关键内部接口（新模块可复用）：`get_db()`（Row factory）、`call_llm_stream(messages, tools, provider_id, provider_key)`、`execute_tool(name, args)`、`get_active_provider()`、`get_or_create_device_id(req, res)`、`estimate_tokens(text)`、`_strip_emoji(text)`、`TOOLS_SCHEMA`、`running_tasks`

DB 表：`sessions` / `messages` / `audit_log` / `attachments` / `todos` / `goals` / `goal_milestones` / `session_summaries` / `provider_models`（2026-09-14 后 `sessions` 多 `kind`/`room_status`/`room_cursor`/`room_pending`，`messages` 多 `speaker`）

---

## 🆕 v2 only-one 完整复盘（2026-09-10）

- **首录**：2026-09-10
- **来源**：v2 only-one v0.3.0-batch-upgrade 完工后的复盘
- **目的**：给未来 v3 / 类似项目参考
- **Skill 文件**：`memory/skills/v2-only-one-retrospective.md`

### 5 个核心结论

1. **先问"跑顺序"再开工**（3 步以上任务必须追问顺序）
2. **AI 主动建议 + 用户拍板**（不是 AI 替用户决定）
3. **端到端测试才能发现真问题**（C 验证发现 kb-pull.sh v1 的 2 个真问题）
4. **secure-token-handling Skill 必须独立**（6 次教训 → 自动加载 → 套模板）
5. **失败是资产不是负债**（10 次失败沉淀 → 未来 v3 不再踩）

### 自我评分：82%（达成 Cherry 80% 目标）

### 时间 / 资源统计

- 12 小时 / 3 天跨度；4 个 commit；~150KB 代码；9 个 tools；4 个 Skill 文件；~15 条 JOURNAL；**10 次失败**（沉淀成资产）

### 关联

- SOUL 观点 F + G（Skill 机制）
- SOUL 观点 H（安全底线 v2 修订 C1）
- v0.3.0-batch-upgrade tag

---

## 🆕 secure-token-handling Skill 抽取 v2（2026-09-10）

- **来源**：6 次 token 教训合并
- **文件**：`memory/skills/secure-token-handling.md`

### 7 次历史教训

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
