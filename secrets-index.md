# 🔐 secrets-index — Agent 高频信息索引（凭据 + 常用资源）

> 路径：`memory/secrets-index.md`
> 用途：AI 知道"用什么 key / 用什么资源 / 在哪 / 怎么用"——**一处索引，避免每次新对话重复问用户**
> **核心规则（本文件对凭据类条目）：只记路径 + 用途 + 变量名，🚫 绝不写明文 key / 密码 / 私钥内容**
> **扩展说明（2026-09-10 起，观点 J）**：本文件已从"纯凭据索引"扩展为"高频信息索引"。非凭据类条目（域名 / ssh 主机 / 端口 / 服务地址）也可收录，字段相同、但可以写明文（这些不是凭据）。

---

## 📡 模型 API Key 索引

| 模型 / 服务 | 路径 | 变量名 | 备注 |
|---|---|---|---|
| MiniMax M3 / M2.7 / M2.5 | 云端 `~/.bashrc` | `ANTHROPIC_AUTH_TOKEN` | 125 字符 sk-cp- prefix（ssh 读）|
| DeepSeek | 主：云端 `/home/ubuntu/only-one/.env`（第 1 行）<br>源：本地 `D:\secrets\deepseek.env`<br>备：SQLite `only-one.db` → `provider_models.deepseek.api_key_enc`（Fernet 加密）| `DEEPSEEK_API_KEY` | 2026-09-12 接入 only-one；sk- 前缀 35 字符；**本地源文件格式是 `api key=sk-xxx` / `api URL=...`（非标准 VAR=value，解析要用 `sk-` 锚点）** |
| DeepSeek（healthCheck 定义）| 云端 `agent-data/webchat/provider-config.json` + `agent-data/tools/provider-config.json` | （无 key，仅端点定义）| healthCheck 用，非暴露 API |
| Gate（量化交易）| 本地 `D:\quant-bot\config\gate.env` | `GATE_API_KEY` / `GATE_SECRET` | chmod 600 |
| OpenAI / Anthropic | （无） | — | 当前未用 |

---

## 🖥️ 云服务器登录

| 名称 | 类别 | 值 / 路径 | 变量名 | 使用场景 | 备注 |
|---|---|---|---|---|---|
| 腾讯云 Ubuntu | ssh 主机 | `ubuntu@152.136.41.22`（S5.2C8G）| — | 远程跑 yt-dlp / ffmpeg / pm2 / docker 等 | — |
| SSH 端口 | ssh 配置 | 22 | — | `ssh -p 22 ...` | 默认端口一般不写 |
| 登录方式 | ssh 认证 | SSH 免密（公钥认证）| — | — | 公钥 `C:\Users\Cryin\.ssh\id_ed25519` |
| MCP 调用方式 | ssh 工具 | `execute-command` 跑 `ssh "CMD"`（双引号）| — | 通过 agent 跑远端命令 | 单引号易炸（见 FACT.md 慎用） |

---

## 🌐 Cherry Studio 配套服务

| 服务 | 路径 | 凭据位置 |
|---|---|---|
| webchat 登录 | 云端 `/home/ubuntu/agent-data/webchat/` | `keys.json`（登录密钥，非 LLM key）|
| band-relay | 云端 `/home/ubuntu/agent-data/band-relay/` | `.env` mode 600（手环已失败，服务暂留）|
| 飞书推送 | 本地 `D:\decision-improvement\config\feishu.env` | `FEISHU_APP_ID` / `FEISHU_APP_SECRET` |

---

## 🍪 抖音 cookies

- 持久化：云端 `~/.douyin_cookies.txt`（30 天过期重导）
- 临时：本地 `D:\tmp\douyin\*.cookies`（用完 `rm`）
- 浏览器导出：Netscape 格式 ≥ 100 bytes

---

## 🚫 严禁（写进 FACT.md 边界）

- ❌ 不要把明文 key / 密码 / SSH 私钥内容写进 JOURNAL / FACT / SOUL / USER
- ❌ 不要把 secrets-index.md 自己备份到任何明文云端 / Git
- ❌ 不要 echo / cat 任何 .env 内容到对话里
- ✅ 要用 key 时 → 自己去对应路径 Read，**不在对话里贴**

---

## 📦 高频信息（非凭据，可写明文）

> 适用：域名 / 服务地址 / 端口 / 常用路径 / 第三方平台账号 / 项目根目录
> 不适用：凭据类（走上面的章节）

| 名称 | 类别 | 值 / 路径 | 变量名 | 使用场景 | 备注 |
|---|---|---|---|---|---|
| （待填充）| — | — | — | — | 用户首次提到时由 AI 建议收录 |

---

## 🆕 收录触发规则（2026-09-10 起，观点 J）

- **被动触发**：同一类信息在对话中**第 2 次出现** → AI 主动问"这个以后常用吗？要收录到 secrets-index.md 吗？"
- **主动建议**：用户**第 1 次提到**明显高频可能信息（域名 / ssh 主机 / 端口 / 服务地址 / 项目根目录）→ AI 主动问"这个看起来以后会常用，要不要顺手收录？"
- **收录由用户拍板**（AI 不偷偷收）
- **下次使用**：AI 主动查本文件 → 不再问用户

---

## 🔄 何时更新

- 新增服务 / key / 高频信息 → 加一行
- 路径变更 → 改对应行
- 旧 key 失效 → 删对应行
- 30 天审视一次（跟 FACT.md 维护规则同步）