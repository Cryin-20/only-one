# FACT.md - Cherry AI 持久知识库

> 最后更新：2026-09-10（v11 — 加"信息冗余削减"触发规则段，源自 SOUL 观点 J）
> 维护规则：6+ 个月重要的写入这里，一次性的写 JOURNAL.jsonl
> **边界**：本文件只装事实/决策/技术参数；AI 行为规则 → 进 Cherry Studio system prompt，不在本文件
> **凭据边界**：本文件 / JOURNAL / SOUL / USER 🚫 严禁明文 API key / 密码 / SSH 私钥内容。凭据路径索引见 `memory/secrets-index.md`

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

### 关键规则

| 行为 | 是否允许 |
|---|---|
| AI 读 .env + 输出长度/前 8/后 4 | ✅ |
| AI 用 SCP 文件模式传 token | ✅ |
| AI 用 SecureString 输入 | ✅ |
| **AI 输出 token 明文** | ❌ 核心防御 |
| AI 用 read_text_file 读 .env | ❌ |

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
