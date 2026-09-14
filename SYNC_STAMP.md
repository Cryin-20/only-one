# 🔄 同步标记（SYNC STAMP）

> **用途**：一眼确认"这份记忆是哪一次同步的快照"。
> **写入方**：只由 **本地 Cherry Studio（Windows）** 写入；云端只读（避免两端同时改导致 git 冲突）。
> **自动维护**：`C:\Users\Cryin\cherry-sync\sync_cherry_memory.ps1` 每次同步时重写本文件的"最后同步时间"并追加一行变更记录。

---

## 当前状态

| 项 | 值 |
|---|---|
| **最后同步时间** | **2026-09-14 11:50 +0800** |
| **内容版本** | FACT v13 ・ JOURNAL 44,493 B ・ secrets-index +DeepSeek 行 |
| **来源** | Cherry Studio (Windows) → GitHub `cherry-memory` |
| **云端位置** | `/home/ubuntu/.cherry/memory/` |
| **推送 commit** | 见 `git log -1 --format='%h %ci %s'` |

---

## 同步链路

```
本地 memory (Windows)  --push-->  GitHub Cryin-20/only-one : cherry-memory  --pull-->  云端 /home/ubuntu/.cherry/memory
```

| 端 | 触发方式 |
|---|---|
| 本地 push | 任务计划 `Mon-Fri 11:50 / 17:00`（`sync_cherry_memory.ps1`）|
| 云端 pull | `cherry-memory-pull.timer`（每 15 分钟）+ `cherry-memory-sync.timer`（Mon-Fri 12:00 / 16:50）|

---

## 包含内容

| 文件 | 说明 |
|---|---|
| `SOUL.md` | AI 人格 / 行为规则 / 观点 A–J |
| `USER.md` | 用户身份简档 |
| `FACT.md` | 持久知识库（6+ 月重要事实 / 决策 / 技术参数）|
| `JOURNAL.jsonl` | 事件流（一次性事件 / 错误 / 坑）|
| `secrets-index.md` | 高频信息索引（凭据只记路径；非凭据可明文）|
| `skills/` | 跨项目可复用工作流 |
| `daily/` | 每日记录 |

---

## 变更记录（最近 10 次）

| 时间 (+0800) | 说明 |
|---|---|
| 2026-09-12 16:46 | **首次引入本标记文件**。同步 FACT v13（only-one v0.3.16「回复说到一半停」根因与修复 + DeepSeek provider 接入）、JOURNAL（今日诊断记录）、secrets-index（DeepSeek key 位置）|
