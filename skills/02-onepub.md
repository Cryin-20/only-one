---
name: onepub
description: Cherry 公众号一键排版 + 发布（md2wechat CLI + 微信素材库 + test-draft 草稿）
when-to-use:
  - 公众号
  - OnePub
  - md2wechat
  - 微信发布
  - 微信公众号
  - TrendPublish
  - Cherry 公众号
created: 2026-09-08
source: FACT.md "🆕 Cherry 公众号一键排版 + 发布工具链 OnePub（2026-09-07 装机）" 章节
related-files:
  - D:\tools\md2wechat-skill\
  - D:\tools\trend-publish\
  - D:\tools\run-trendpublish-real-publish.ps1
  - D:\tmp\runtiancui\
---

# OnePub — Cherry 公众号一键排版 + 发布

## 目标
从本地 Markdown 源稿 → md2wechat 排版 → 上传素材库 → 创建 test-draft 草稿 → 用户手动在公众号后台点群发。**AI 只能帮到草稿箱，群发必须用户手动**（不可逆操作）。

## 前置条件
- `md2wechat.exe` 已装：`D:\tools\md2wechat-skill\md2wechat.exe`
- 全局配置 `C:\Users\Cryin\.config\md2wechat\config.yaml`（AppID 已配 + Secret 走 env）
- **凭据**：AppSecret 在 `D:\secrets\wechat.env`，env `WECHAT_SECRET` 注入（**不走对话明文**）
- **AI 生图**（可选）：MiniMax image-01 endpoint + Doubao Seedream 5.0 lite model ID `doubao-seedream-5-0-260128`
- **真发服务器 IP 白名单**：`113.110.214.238`（本机动态 IP，重启路由器会变）

## 操作步骤（5 步 — 双工具链）

### 路径 A：手动排版 + 发布（md2wechat 单工具，AI 主导内容）

```bash
# 1) AI 写产品稿（参考 v6 教训：风格 + 受众 + 调性先对齐）
# 2) md2wechat preview 生成 HTML 设计 prompt → AI 据此写 HTML
md2wechat.exe preview --md "D:\tmp\xxx.md"

# 3) 上传图片到微信素材库（用 .NET Process 注入 WECHAT_SECRET，不打印明文）
powershell -File D:\tmp\runtiancui\upload-images.ps1

# 4) 替换 HTML 本地 src 为微信 URL
powershell -File D:\tmp\runtiancui\replace-and-test-draft.ps1

# 5) md2wechat test-draft 创建草稿（返回 media_id）
#    → 用户手动在公众号后台点群发
```

### 路径 B：全自动资讯流水线（TrendPublish）

```bash
# 1) 配置数据源（gdelt/hackernews/arxiv 都是 search provider，要 "search:关键词" 前缀）
# 2) 跑 workflow（必须传 --force-publish flag，否则 publish-result 仍是 dry-run）
powershell -File D:\tools\run-trendpublish-real-publish.ps1
```

## 关键坑（v7 累计 18+ 个，最常踩的 6 个）

1. **md2wechat test-draft 名字误导**：名字是 "test" 但**实际创建真草稿**（不是 dry-run）— 草稿 ≠ 群发，草稿可逆，群发不可逆
2. **TrendPublish LLM endpoint 错配**：走 OpenAI Chat Completions → `https://api.minimaxi.com/v1/chat/completions`（**不是** `/anthropic/chat/completions`）
3. **MiniMax key 错配**：`MINIMAX_CN_API_KEY` (hermes) 只用于 Anthropic Messages API；**`MINIMAX_API_KEY` (gate.env) 用于 OpenAI Chat Completions + 图片生成** — 测得 401 vs 200
4. **Doubao Seedream 5.0 lite model ID 错**：GitHub 例子 `doubao-seedream-5-0-lite-250428` ❌ → 官方 `doubao-seedream-5-0-260128` ✅
5. **AI 图像默认带 watermark**：Seedream 5.0 lite watermark 默认 true（带"AI生成"水印）。修：API body 加 `"watermark": false`
6. **AI 图像 prompt 触发自生文字**：prompt 没强调"no text"时 AI 自带中文/英文标语。修：prompt 加 "IMPORTANT: no text, no words, no letters, no characters, no logos, no watermarks"

## 写产品稿 4 条硬规则（v6 沉淀）
1. **AI 写产品文案前必须先确认：风格 + 受众 + 调性**（3 选 1）
2. **AI 写产品文案配图必须先想好**（不能 afterthought）
3. **AI 抓产品数据先去京东/天猫 item 页**，不去百度百科（百度百科是辅助）
4. **第三方图库的版权声明图不能用**（如昵享网明确禁止"商用"）

## 慎用 / 反例（SOUL 观点 F 配套）
- ❌ **不要 AI 自动群发到粉丝**（不可逆，必须用户手动在公众号后台点"群发"）
- ❌ **不要用 AI 生成的产品稿无人工审**（v7 润田翠"一般般"教训 — AI 故事化不够）
- ❌ **不要把 AppSecret 明文贴到对话**（v7 教训 — 走 .NET Process 隔离 + env 注入）
- ❌ **不要在 IP 白名单变了后没更新**（生产建议部署云端固定 IP）
- ❌ **不要用大尺寸 AI 生图（如 1024x1024）— Seedream 5.0 lite 不支持**（最小 2048x2048）

## 输出位置
- 本地 HTML：`D:\tools\md2wechat-skill\xxx.html`
- 微信素材库：永久（media_id 永久有效，URL 24h 过期）
- 草稿箱：公众号后台 → 草稿箱 → 待群发

## 关联（wikilink）
- [[🆕 Cherry 公众号一键排版 + 发布工具链 OnePub（2026-09-07 装机）]]（FACT.md 主章节）
- [[🆕 AI 写产品文案的 6 次返工反思（2026-09-08 沉淀）]]（v6 4 条硬规则来源）
- [[🆕 润田翠 v4 公众号产品稿全链路实战（2026-09-08）]]（v7 完整实战 — 5 脚本版本 + 7 个新坑）
- [[🚨 安全底线原则]]（AppSecret 泄漏必须重置 — v7 Doubao key 触发）
- [[🆕 抖音视频一键抓取 + 分析工具 OneTable]]（兄弟工具链 — 视频方向）

## 下次更新触发
- md2wechat 新版本 / 新 CLI flag → 更新路径 A
- TrendPublish 改 SQLite seed 同步逻辑 → 加新坑
- 公众号 API 新增能力（如图片智能裁剪）→ 路径 A 升级
- 润田翠 v5/v6 实战完成 → 把新坑加进"关键坑"
