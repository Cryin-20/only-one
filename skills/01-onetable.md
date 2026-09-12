---
name: onetable
description: 抖音视频一键抓取 + 转写 + 分析（Edge cookies → yt-dlp → 云端 faster-whisper → 拉回本地）
when-to-use:
  - 抖音
  - 视频抓取
  - OneTable
  - yt-dlp
  - douyin
  - faster-whisper
  - 抖音视频分析
created: 2026-09-08
source: FACT.md "🆕 抖音视频一键抓取 + 分析工具 OneTable（2026-09-06 重构命名 v3）" 章节
related-files:
  - D:\tools\OneTable.ps1
  - D:\tools\douyin-fetch.ps1
  - D:\tmp\douyin\
---

# OneTable — 抖音视频一键抓取 + 分析

## 目标
从抖音分享链接（v.douyin.com / www.douyin.com）一键抓取视频 → 转音频 → faster-whisper 转写 → 拉回本地 ready for AI 总结。

## 前置条件
- Edge 已登录 douyin.com（cookies 提取前提）
- 云服务器 `152.136.41.22` 已装 yt-dlp + ffmpeg + faster-whisper-small
- SSH key: `C:\Users\Cryin\.ssh\id_ed25519`
- OneTable.ps1 文件**必须有 UTF-8 BOM**（v7 关键坑，PowerShell 5.1 ANSI 解 UTF-8 无 BOM → 中文乱码 → 解析失败）

## 操作步骤（3 条路径，按需选）

### 路径 A：OneTable.ps1 一键（推荐，cookies 自动提取）
```powershell
powershell -File D:\tools\OneTable.ps1 -Url "https://v.douyin.com/xxxxx/"
```
完整 8 步（自动）：关 Edge → 提 cookies → 重启 Edge → yt-dlp 下载 → scp 上传 → ssh 云端 analyze → scp 拉回 → 完成。产物在 `D:\tmp\douyin\<video_id>\`。

### 路径 B：douyin-fetch.ps1（手动 cookies 备选）
需先 Chrome 扩展 `Get cookies.txt LOCALLY` 导出 douyin Netscape 格式 cookies → `D:\tmp\douyin\cookies.txt`：
```powershell
powershell -File D:\tools\douyin-fetch.ps1 -Url "<url>" -CookiesFile "D:\tmp\douyin\cookies.txt"
```

### 路径 C：firecrawl 只拿元数据 + AI 摘要（最快，5-15 秒）
```python
firecrawl_scrape("<v.douyin.com/xxx>")
# 拿：真实 URL + 5 章节概要 + AI 自动总结 + 互动数据
# 适合：快速认知扩展 / 不需要原文转写
```
**AI 失误风险**：firecrawl 给的是 AI 摘要，可能漏关键细节（参考 v7 润田翠文案教训）

## 关键坑（v7 累计 18+ 个，最常踩的 5 个）

1. **OneTable.ps1 编码问题**：UTF-8 无 BOM 文件被 PowerShell 5.1 ANSI 解码 → 中文乱码 → 解析失败。修：
   ```powershell
   [System.IO.File]::WriteAllText($path, $content, [System.Text.UTF8Encoding]::new($true))
   ```
2. **execute-command 只能跑 single-token 命令**：带 args 必 ENOENT。修：写 .ps1/.sh 脚本 + `powershell -File` 或 `bash /path/to.sh`
3. **.NET Process 同步 ReadToEnd() 长跑 deadlock**：child buffer 满 → child 阻塞写 → parent 阻塞读。修：`BeginOutputReadLine + OutputDataReceived` 异步事件 + 不调 ReadToEnd
4. **pwsh sandbox 下 tasklist/where.exe 不可用**：用 `Get-Process` / `Get-Command` 代替
5. **Edge 关掉影响用户当前使用**：OneTable 自动重启 Edge + 标签页恢复，但中间 ~5 秒会断网

## 输出位置
`D:\tmp\douyin\<video_id>\`：
- `<video_id>.mp4` — 视频
- `<video_id>.wav` — 音频（16kHz mono）
- `<video_id>_segments.jsonl` — faster-whisper 转录（带时间戳）
- `<video_id>_keyframe_*.jpg` — 关键帧（每30秒一张）

## 慎用 / 反例（SOUL 观点 F 配套）
- ❌ 不要用来抓 18+/违规内容（cookies 会被 douyin 风控 + 触发限流）
- ❌ 不要大批量抓（>10 个/小时）— 触发 douyin 限流 + IP 风控
- ❌ 不要把 cookies 文件传到云端持久化（OneTable 自动清理，但手动路径要小心）
- ❌ 不要在 Edge 关掉后立即重新打开（cookies DB 锁未释放，会导致提取失败）
- ❌ 不要假设 firecrawl 摘要 = 视频原文（AI 摘要会丢细节）

## 关联（wikilink）
- [[🆕 抖音视频一键抓取 + 分析工具 OneTable]]（FACT.md 主章节）
- [[🆕 Cherry 公众号一键排版 + 发布工具链 OnePub]]（兄弟工具链 — 公众号方向）
- [[关键 bug 修复]]（v7 累计 18+ 个 OneTable 坑）

## 下次更新触发
- OneTable.ps1 改路径 / 改步骤 → 更新路径 A
- 抖音防爬机制升级（如强制 WebAuthn）→ 加新坑
- firecrawl 支持视频原文转写 → 路径 C 升级
