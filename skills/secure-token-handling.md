---
name: secure-token-handling
description: AI 处理 token / password / API key / secret 的安全操作模板
when-to-use: ["token", "PAT", "API key", "secret", "password", "凭据", "credential", "GitHub PAT", "MiniMax", "ssh key", "secure"]
created: 2026-09-10
source: FACT.md "🆕 secure-token-handling Skill（2026-09-10 抽取）"
related-files:
  - "D:\\projects\\pm-demo\\only one\\scripts\\kb-pull.sh"
  - "D:\\tmp\\only-one-deploy\\step6-inject-secure.py"
  - "C:\\Users\\Cryin\\AppData\\Roaming\\CherryStudio\\Data\\Agents\\6c6b9e7b-a53a-40ec-919a-84b4f4f34293\\SOUL.md (viewpoint H)"
---

# secure-token-handling — Token 处理安全模板

## 目标
AI 跑 token / password / API key / secret 相关操作时，**不输出明文**到 console / 对话 / file，避免 token 永久泄漏。

## 前置条件

- 凭据已存到 secrets-index 路径（`memory/secrets-index.md` 索引）
- AI 知道凭据的**路径** + **用途**，但**不读明文**（除 SOUL 观点 H C1 允许的情况）

## 3 条核心约束（不可破）

按 SOUL 观点 H（🛡 安全底线 v2 修订）：

| 行为 | 是否允许 | 原因 |
|---|---|---|
| AI 用 python 读 .env + 输出长度/前 8/后 4 | ✅ 允许 | C1 修订（用户拍板）|
| AI 用 SCP 文件模式传 token | ✅ 允许 | token 不进 command line |
| AI 用 SecureString 接收输入 | ✅ 允许 | 输入隐藏 |
| **AI 在 console / 对话 / file 输出 token 明文** | ❌ **禁止** | **核心防御，不可删** |
| AI 用 read_text_file / cat / Get-Content 读 .env | ❌ **禁止** | 工具 hardcoded 全文输出 |
| AI 把 token 拼到 URL 写入 git config | ❌ **禁止** | git config 永久保存 |

## 操作步骤（按场景选）

### 路径 A：python 读 .env（诊断场景）

```python
import os
ENV_PATH = r"D:\secrets\xxx.env"
token = None
with open(ENV_PATH, encoding='utf-8') as f:
    for line in f:
        if line.startswith('TOKEN_NAME='):
            token = line.split('=', 1)[1].strip()
            break

# ✅ 安全：只输出元信息
print(f"Length: {len(token)} chars")
print(f"Prefix: {token[:8]}...")
print(f"Suffix: ...{token[-4:]}")

# ❌ 禁止：print(token) 或 print(f"Token: {token}")
del token  # 立即清内存
```

### 路径 B：SecureString 输入（Win PowerShell）

```powershell
# SecureString 模式（输入隐藏）
$secure = Read-Host -AsSecureString -Prompt "GitHub PAT"
$BSTR = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secure)
$token = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto($BSTR)
[System.Runtime.InteropServices.Marshal]::ZeroFreeBSTR($BSTR)
$env:KB_TOKEN = $token
# 跑完清理
Remove-Item Env:KB_TOKEN
```

### 路径 C：SCP 文件传 token（最稳，跨 Win/Linux）

```python
import subprocess, tempfile, os
token = "..."  # 从 .env 读（按路径 A）
# 写本地临时文件
tmp = tempfile.NamedTemporaryFile(mode='w', delete=False, encoding='utf-8', suffix='.txt')
tmp.write(token); tmp.close()
# SCP 到云端
subprocess.run(["scp", tmp.name, f"user@host:/tmp/.token_tmp"], capture_output=True, timeout=30)
# 立即删本地
os.remove(tmp.name)
del token
# SSH 跑脚本（脚本读 /tmp/.token_tmp，不 echo）
```

### 路径 D：GIT_ASKPASS + git credential helper（云端 kb-pull）

**用在** kb-pull.sh 等云端定时任务脚本：

```bash
#!/bin/bash
# 1) 生成临时 askpass 脚本（chmod 700，结束后删）
ASKPASS_SCRIPT="/tmp/.kb_askpass"
cat > "$ASKPASS_SCRIPT" << EOF
#!/bin/bash
echo "\$GITHUB_TOKEN"
EOF
chmod 700 "$ASKPASS_SCRIPT"

# 2) 用 askpass（token 不进 .git/config）
export GIT_ASKPASS="$ASKPASS_SCRIPT"
export GIT_TERMINAL_PROMPT=0

# 3) git 操作（fetch / clone / pull）
git fetch origin main

# 4) 清理
rm -f "$ASKPASS_SCRIPT"
unset GIT_ASKPASS GIT_TERMINAL_PROMPT
```

**关键**：**不要** `git remote set-url origin "https://x-access-token:$TOKEN@..."`（这会把 token 写进 .git/config）

## 关键坑（6 次踩坑累计）

### 坑 1: read_text_file / cat / Get-Content 输出明文

**症状**：用 `read_file` 或 `cat xxx.env` → 工具 hardcoded 全文输出 → token 进对话历史  
**修复**：用 python 读 + 只输出长度/前 8/后 4

### 坑 2: PowerShell Read-Host 显示密码

**症状**：`Read-Host -Prompt` 默认不隐藏 → 输入可见 → 旁观者看见 token  
**修复**：用 `-AsSecureString` + `SecureStringToBSTR` + `ZeroFreeBSTR`

### 坑 3: Write-Host 输出 token 明文

**症状**：`Write-Host "FULL_TOKEN: $token"` → token 进 Win console 历史  
**修复**：用 `${#TOKEN}` 输出长度，或 `Write-Host "Length: $($token.Length)"`

### 坑 4: read_text_file / get_file 工具 hardcoded 全文

**症状**：MCP 工具类（read_text_file 等）直接返回文件全文，没法"部分读"  
**修复**：用 python open() + 只 print 元信息

### 坑 5: kb-pull.sh 把 token 存到 .git/config

**症状**：`git remote set-url origin "https://x-access-token:$TOKEN@..."` → token 进 .git/config 永久保存  
**修复**：用 `GIT_ASKPASS` 环境变量（git config 不存）

### 坑 6: SSH stdin pipe 传 token（CRLF 问题）

**症状**：Win Python 写出的字符串是 `\r\n`，云端 bash 不认 → `$'\r': command not found`  
**修复**：用 SCP 文件模式（路径 C）替代 SSH stdin

### 坑 7（预防）：AI 跑命令前必须自检

**症状**：跑 `git remote -v` → 命令**必然**输出 token URL → 又一次泄漏  
**修复**：跑命令**前**先问"输出含敏感数据吗" → 含 → 用 sed 过滤再 echo：

```bash
git remote -v | sed 's|x-access-token:[^@]*@|x-access-token:REDACTED@|'
git config --list | grep url | sed 's|x-access-token:[^@]*@|REDACTED@|'
```

### 坑 8（基础设施）：cloudflared quick tunnel 后台跑必须 setsid

**症状**：纯 `nohup cloudflared ... &` → SSH session 退出后 cloudflared 被 kill → tunnel 失效  
**症状 2**：默认 stdout buffering → 30 秒后才看到 trycloudflare URL（实际 stderr 立即有）  
**修复**：用 `setsid` + 重定向 stdin + stderr 也合并：

```bash
# ✅ 正确（云端后台稳定跑）
nohup setsid cloudflared tunnel --url http://localhost:8099 > /tmp/cf.log 2>&1 < /dev/null &
sleep 30
grep 'trycloudflare' /tmp/cf.log | head -3

# ❌ 错误（SSH 退出就 kill / URL 不出来）
nohup cloudflared tunnel --url http://localhost:8099 > /tmp/cf.log 2>&1 &
```

### 坑 9（基础设施）：trycloudflare URL 在 stderr 不是 stdout

**症状**：用 `2>&1 | head -3` 才能抓到 trycloudflare URL，纯 `head -3` 拿不到  
**修复**：cat 日志文件 = 包含 stderr（因为用 `2>&1` 重定向），或运行时直接 `2>&1`

## 慎用 / 反例（SOUL 观点 F 配套）

### ❌ 反例清单（不要做的事）

```bash
# ❌ 反例 1: cat 输出明文
cat ~/.env
cat /home/ubuntu/.env-github

# ❌ 反例 2: echo 整个变量
echo "$TOKEN"
echo "Token is: $TOKEN"

# ❌ 反例 3: write-host 明文
Write-Host "Token: $env:KB_TOKEN"
Get-Content .env | Select-String "TOKEN="  # 输出含明文

# ❌ 反例 4: 命令输出含 token 不过滤
git remote -v  # URL 里含 token
git config --list | grep url

# ❌ 反例 5: 把 token 拼到 URL 写 git config
git remote set-url origin "https://x-access-token:$TOKEN@github.com/xxx.git"

# ❌ 反例 6: 把 token 拼到 SSH cmdline
ssh ubuntu@host "echo $TOKEN > /path/to/file"

# ❌ 反例 7: 用 read_text_file / Get-Content 工具读 secrets
read_file("D:/secrets/xxx.env")  # 工具 hardcoded 全文
```

### ✅ 正确示例

```bash
# ✅ 输出长度/前缀/后缀
echo "Length: ${#TOKEN}"
echo "Prefix: ${TOKEN:0:8}"
echo "Suffix: ...${TOKEN: -4}"

# ✅ git remote 用 sed 过滤
git remote -v | sed 's|x-access-token:[^@]*@|x-access-token:REDACTED@|'

# ✅ python 读 + 输出元信息
python -c "
with open('.env') as f:
    for line in f:
        if line.startswith('TOKEN='):
            token = line.split('=',1)[1].strip()
            print(f'Length: {len(token)}')
            print(f'Prefix: {token[:8]}')
            del token
"

# ✅ SecureString 输入
Read-Host -AsSecureString -Prompt "Token"

# ✅ SCP 文件传 token
scp token.txt user@host:/tmp/.token_tmp && rm token.txt

# ✅ GIT_ASKPASS 不存 token
export GIT_ASKPASS=/tmp/.askpass_script
```

## 6 次历史教训表

| # | 时间 | 事件 | 根因 | 修复 |
|---|---|---|---|---|
| 1 | 2026-09-08 | Doubao Seedream key 进对话 | `read_file` 显示全文 | 改用 python 长度输出 |
| 2 | 2026-09-10 | PowerShell `Read-Host -Prompt` 显示密码 | 默认不隐藏 | 改用 `-AsSecureString` |
| 3 | 2026-09-10 | GitHub PAT 明文进 console | `Write-Host 'FULL_TOKEN:'` | 改用 length + prefix + suffix |
| 4 | 2026-09-10 | `read_text_file` 输出 token 明文 | 工具 hardcoded 全文 | 改用 python 读 .env |
| 5 | 2026-09-10 | kb-pull.sh 把 token 存到 .git/config | `git remote set-url` 永久存 | 改用 GIT_ASKPASS |
| 6 | 2026-09-10 | `git remote -v` 输出 token 明文 | AI 没自检命令输出 | 用 sed 过滤 + AI 跑前自检 |

## 输出位置

- 本地（agent-data-dir）：`C:\Users\Cryin\AppData\Roaming\CherryStudio\Data\Agents\<id>\memory\skills\secure-token-handling.md`（**标准路径**，Cherry Studio 加载）
- 本地（fallback）：`D:\projects\pm-demo\only one\memory\skills\secure-token-handling.md`
- 云端 v2：`/home/ubuntu/.cherry/memory/skills/secure-token-handling.md`

## 关联（wikilink）

- SOUL 观点 H（🛡 安全底线 v2 修订 C1）
- FACT.md "🆕 secure-token-handling Skill（2026-09-10 抽取）"
- v1 凭据路径索引（memory/secrets-index.md）
- v2 only-one v0.3.0-batch-upgrade tag
- 6 条历史 JOURNAL（security-leak + security-powershell-readhost + soul-rule-change + security-policy-violation + kb-pull-retry-needed + security-policy-violation-2）

## 下次更新触发

满足任一条件即触发更新：
1. **同类问题再出现**（token 泄漏第 7 次）→ append 新坑到"关键坑"段
2. **新传输方式**（如 git credential helper、SSH key、OAuth）→ append 新路径
3. **新工具/平台**（如用 Claude / GPT 等其他 AI）→ 更新"反例清单"
4. **SOUL 观点 H 修改** → 同步更新"3 条核心约束"
5. **新场景**（如 VPN / 容器化 / WSL）→ append "前置条件"段

## 测试场景（验证 Skill 有效性）

| 场景 | 测试方法 | 期望结果 |
|---|---|---|
| 1 | AI 跑 `cat .env` | Skill 阻止 → AI 改用 python 路径 A |
| 2 | 用户说"我看了 token 是 xxxx" | Skill 提醒用户**不要**贴明文到对话 |
| 3 | AI 跑 `git remote -v` | Skill 提示"输出含 token" → AI 用 sed 过滤 |
| 4 | AI 写 kb-pull.sh v3 | Skill 提供 GIT_ASKPASS 模板 |
| 5 | 新 AI agent 加载 Skill | 触发词命中 → 自动加载 |

---

**Skill 抽取时间**：2026-09-10  
**Skill 抽取原因**：6 次 token 教训（SOUL 观点 G "≥3 次" 强烈触发）  
**未来维护人**：Cryin（v1 Cherry 训诫 + v2 only-one 实战）  
**Skill 状态**：v1（首次抽取）
