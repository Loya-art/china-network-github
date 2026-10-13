---
name: china-network-github
description: "在中国大陆网络环境下操作 GitHub 仓库 — 镜像源选择、API 直连替代、git clone/fetch 绕过方案、更新检查"
version: 1.2.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [GitHub, Mirror, Proxy, China, Network, Git, ghproxy]
    related_skills: [github-repo-management, github-auth, github-pr-workflow]
---

# GitHub 国内网络操作指南

在中国大陆网络环境下，GitHub（尤其是 `github.com`、`raw.githubusercontent.com`）通常被限制直连。
本技能覆盖 clone、fetch、版本检查、更新等操作的替代方案。

## 环境判断

GitHub 被墙的典型表现：

```bash
# git clone/fetch 超时
timeout 5 git ls-remote https://github.com/owner/repo.git
# → 超时 / 无响应 / Connection refused

# GitHub API 通常仍可用
curl -sI --connect-timeout 5 "https://api.github.com"
# → HTTP/1.1 200 OK 或 403 (rate limited) 或 301
```

## 核心原则

1. **`api.github.com` REST API 时通时不通，每次先测再用** — 2026-07 实测直连可用；2026-08 实测直连 000（含 raw.githubusercontent.com）。`curl -sI --max-time 8 "https://api.github.com"` 先探活，不要凭旧记录直接依赖
2. **`ghproxy.net` 是主力镜像** — HTTP 下载可用，但 git remote 偶尔超时
3. **Git clone/fetch 被墙时，优先通过 API 获取信息**（版本号、文件列表），不需要每次都走 git
4. **`registry.npmjs.org` 国内直连通常稳定可用 → 当 GitHub 情报源**：查包名变体（`<tool>`, `<tool>-cli`, `<tool>-sdk`, `@<scope>/cli`），`repository` 字段直接暴露 GitHub 组织/仓库；下载 tarball（`https://registry.npmjs.org/<pkg>/-/<pkg>-<ver>.tgz`）解压可读仓库内文件（README、package.json 等），全程不碰 GitHub
5. **Chromium 浏览器（Chrome/Edge）在 Windows 上不接受后台键盘事件**（`Chrome_WidgetWin_1` 窗口类限制）— 需要 PowerShell SendKeys 或 `set_value` 加用户按回车组合操作（见 `references/chromium-windows-automation.md`）
6. **需要代理才能买代理** — 部分机场网站本身被墙，需要用其他方法先搞到临时通道（见下方方案）
7. **Clash 节点状态要先测**：`curl -s --proxy http://127.0.0.1:7897 --max-time 10 -o /dev/null -w "%{http_code}" https://www.google.com` → 000 即节点挂（此时国外全挂但国内直连正常，症状像"所有国外站都死了"）。Clash 规则：国内域名直连、国外走节点
8. **开系统代理会影响 browser 工具的出口**：`reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable /t REG_DWORD /d 1 /f` 可让 browser 工具访问被分流到节点而节点挂掉的站（走规则直连）；**用完必须复位 /d 0**，否则节点挂了时用户浏览器国外站全挂

## 方案：Watt Toolkit（Steam++）加速通道 — 本机已有加速时的最优解

当用户机器装有 Watt Toolkit（Steam++，UWP 应用，App 列表可见）且 GitHub 加速已配置过时：

**原理**：Watt Toolkit 把 GitHub 域名写入 hosts（`127.0.0.1 github.com` / `api.github.com` / `raw.githubusercontent.com` 等），并监听 `0.0.0.0:443` 用**自签证书 MITM** 转发。进程名：`Steam++.exe` + `Steam++.Accelerator.exe`。

**启用步骤**（如果加速服务没在运行）：
```bash
# 1. 检查 hosts 是否已被改写
grep -i "github" /c/Windows/System32/drivers/etc/hosts
# 2. 检查 443 是否监听
netstat -ano | grep -E ':443 .*LISTENING'
# 3. 没监听则启动 Watt Toolkit（UWP 应用名 4651ED44255E.47979655102CE_k6txddmbb6c52）
powershell -Command "Start-Process 'shell:AppsFolder\4651ED44255E.47979655102CE_k6txddmbb6c52!App'"
sleep 5
# 4. 验证：Steam++.Accelerator.exe 在跑 + 443 监听
```

**使用要点（关键坑）**：
- ⚠️ **curl 必须 `-k`**（自签证书 MITM，不 -k 直接 SSL 握手失败）
- ⚠️ **git clone 走不通**：schannel 和 openssl 后端都会失败（`schannel: failed to receive handshake` / `TLS connect error: unexpected eof`），即使 `http.sslVerify=false`。**用 curl -k + GitHub API 替代 git**：
```bash
# 拿文件树
curl -sk "https://api.github.com/repos/owner/repo/git/trees/main?recursive=1"
# 逐文件下载（raw.githubusercontent.com 也在加速列表）
curl -sk "https://raw.githubusercontent.com/owner/repo/main/path/file" -o local_file
```
- ✅ `api.github.com`、`github.com`、`raw.githubusercontent.com` 都被加速；`codeload.github.com`（zipball/archive 下载）**不在加速列表**，302 后必失败
- ✅ `-c http.sslVerify=false` 对 git 无效（握手阶段就挂），别浪费时间
- Watt Toolkit 关闭后 hosts 残留 127.0.0.1 映射但 443 无服务 → ERR_CONNECTION_REFUSED

## 方案：检查 git 仓库是否最新（无需 fetch）

```bash
LOCAL_TAG=$(cd /path/to/repo && git describe --tags 2>/dev/null)
LATEST=$(curl -sL "https://api.github.com/repos/owner/repo/releases/latest" \
  -H "User-Agent: Mozilla/5.0" \
  | python3 -c "import sys,json; r=json.load(sys.stdin); print(r.get('tag_name',''))")
if [ "$LOCAL_TAG" = "$LATEST" ]; then
  echo "已经是最新版本"
else
  echo "本地: $LOCAL_TAG  远程: $LATEST  可更新"
fi
```

## 方案：git clone 通过镜像

```bash
# 方式 1: 通过 ghproxy（最常用）
git clone https://ghproxy.net/https://github.com/owner/repo.git

# 方式 2: 如果 ghproxy 不可用
# 先通过 API 下载 release zip（如果是 release）
curl -L -o repo.zip "https://api.github.com/repos/owner/repo/zipball/latest"

# 方式 3: 使用 Python urllib（当 curl -o 报错时）
python3 -c "
import urllib.request, json, ssl, os
ctx = ssl._create_unverified_context()
# 查 latest release
url = 'https://api.github.com/repos/owner/repo/releases/latest'
resp = urllib.request.urlopen(url, timeout=20, context=ctx)
data = json.loads(resp.read())
# 下载 zipball
zip_url = data['zipball_url']
urllib.request.urlretrieve(zip_url, 'repo-latest.zip')
print('Downloaded:', os.path.getsize('repo-latest.zip'), 'bytes')
"
```

### ⚠️ GitHub Release 二进制下载：镜像实测记录

**2026年7月实测** — GitHub API (`api.github.com`) 通常可达，但 release binary 直接下载（`curl -L "https://github.com/.../releases/download/..."`）**失败（exit code 56）**，且以下镜像全部失效：

- `ghproxy.net` ❌
- `mirror.ghproxy.com` ❌
- `ghproxy.com` ❌
- `github.petly.top` ❌

**可行替代路径（绕过 GitHub binary 下载的限制）：**

| 方法 | 适用场景 | 实测结果 |
|------|---------|:--------:|
| `winget install --id <PackageId>` | Windows 原生工具 | ✅ LibreOffice, poppler, jq, MiKTeX, 1Password CLI |
| `npm install -g <pkg>` | Node.js CLI 工具（npm 仓库不受 GitHub 封锁影响） | ✅ claude-code, codex, opencode-ai |
| `uv tool install <pkg>` | Python CLI 工具（无需 C++ Build Tools） | ✅ manim, comfy-cli |
| `uv pip install <pkg>` | Python 库（uv 比 pip 更擅长 wheel 解析） | ✅ 成功安装 manim 及其依赖 |
| `choco install <pkg>` | 备选 Windows 包管理（但需管理员权限） | ⚠️ 可能因权限失败 |

**适用场景判断：** 当需要安装一个 CLI 工具但 GitHub 直连和镜像均失败时，先检查：
1. 该工具是否有官方 npm/winget/choco 包
2. 如果是 Python 包，用 `uv tool install` 替代 `pip install`
3. 如果是 Windows 工具，用 `winget install`

## 方案：不依赖 git 的完整仓库下载（树 API + raw 直连）

当 `git clone` 走镜像/代理都报 `schannel: failed to receive handshake` 时（2026-07 实测：
本机 Clash 代理下 git clone 直连 github.com 也 SSL 失败），可完全绕过 git，
用 API 列文件 + raw 逐个下载：

```bash
# 1. 列出仓库全部文件（api.github.com 通常可直连）
curl -s "https://api.github.com/repos/owner/repo/git/trees/main?recursive=1" \
  | python -c "import json,sys; [print(i['path']) for i in json.load(sys.stdin)['tree']]"

# 2. raw.githubusercontent.com 通常也可直连（无需代理/镜像）
curl -s -o local_file "https://raw.githubusercontent.com/owner/repo/main/path/to/file"

# 3. 批量下载整个仓库：循环 curl raw，保持目录结构
```

要点：
- `api.github.com` 与 `raw.githubusercontent.com` 的可达性是**浮动的**（2026-07 直连可用，2026-08 直连 000），
  用前先 `curl -sI --max-time 8 "https://api.github.com"` 探活；不可用时改用 npm registry 情报法（见核心原则 4）
- 走 Clash 代理（127.0.0.1:7897）时 api 请求会超时/空响应（节点挂时）
- 下载 tarball（`/archive/refs/heads/main.tar.gz`）通常被墙/失败，优先用 raw 逐文件方案
- 文件多时写成循环脚本即可，单文件 curl 足够快

## 方案：更新已 clone 的仓库

```bash
# 先用 ghproxy 临时改 remote
git remote set-url origin https://ghproxy.net/https://github.com/owner/repo.git
git fetch --tags --depth 1
# 成功后改回
git remote set-url origin https://github.com/owner/repo.git

# 如果 ghproxy git remote 还是超时
# 用 API 下载最新 zip 替换
LATEST_ZIP=$(curl -sL "https://api.github.com/repos/owner/repo/zipball/main" \
  -H "User-Agent: Mozilla/5.0" -o /tmp/repo-latest.zip)
```

## 方案：通过 GitHub API 创建仓库并推送（无需 gh CLI）

当 gh CLI 未安装且不想安装时，可以通过 `api.github.com` REST API 直接完成**创建仓库 → 推送文件**的完整流程，无需任何 git 操作。

### ⚠️ Windows git-bash 坑：curl + 中文 JSON 体报错

在 Windows git-bash/MSYS 环境下，curl 的 `-d` 参数包含**中文字符**时会报 `"Problems parsing JSON"`：

```bash
# ❌ 失败 —— git-bash 对中文 JSON 的 shell 转义有问题
curl -s -X PATCH \
  -H "Authorization: token $GITHUB_TOKEN" \
  -d '{"description":"蒸馏版马克思视角"}' \
  https://api.github.com/repos/owner/repo
# → {"message": "Problems parsing JSON"}
```

同样，`curl -d @file.json` 也失败（如果文件路径含中文）。

**解法：使用 `execute_code` + Python `urllib` 替代 curl**

`execute_code` 运行在 Python 环境下，不受 git-bash shell 转义影响，可直接发送带中文的 JSON：

```python
# execute_code 中调用，中文 JSON 正常工作
import urllib.request, json

data = json.dumps({
    "description": "蒸馏版卡尔·马克思思维视角 —— 历史唯物主义分析框架"
}).encode('utf-8')

req = urllib.request.Request(
    "https://api.github.com/repos/owner/repo",
    data=data,
    headers={
        "Authorization": "token YOUR_GITHUB_TOKEN",
        "Content-Type": "application/json",
    },
    method="PATCH"  # POST=创建, PATCH=更新
)
with urllib.request.urlopen(req) as resp:
    r = json.loads(resp.read().decode())
    print(r.get("description"))
```

在 terminal 中运行则需要写 Python 脚本文件再执行，或在 `python -c "..."` 中用 base64 编码规避中文转义问题。

以下是完整的 Python urllib 创建仓库方案：

```python
import urllib.request, json, ssl, base64

ctx = ssl._create_unverified_context()
headers = {
    'User-Agent': 'Mozilla/5.0',
    'Authorization': 'token YOUR_GITHUB_TOKEN',
    'Accept': 'application/vnd.github.v3+json'
}

# 1. 创建仓库
repo_data = json.dumps({"name": "repo-name", "description": "...", "private": False, "auto_init": False}).encode()
req = urllib.request.Request('https://api.github.com/user/repos', data=repo_data, headers=headers, method='POST')
resp = urllib.request.urlopen(req, timeout=15, context=ctx)
repo = json.loads(resp.read())

# 2. 获取默认分支的 HEAD commit
url = f"https://api.github.com/repos/{owner}/{repo['name']}/git/refs/heads/main"
req = urllib.request.Request(url, headers=headers)
base_sha = json.loads(urllib.request.urlopen(req, timeout=10, context=ctx).read())['object']['sha']

# 3. 创建 blob（文件内容）
blob_data = json.dumps({"content": base64.b64encode(b"file content").decode(), "encoding": "base64"}).encode()
req = urllib.request.Request(f"https://api.github.com/repos/{owner}/{repo['name']}/git/blobs", data=blob_data, headers=headers, method='POST')
blob = json.loads(urllib.request.urlopen(req, timeout=10, context=ctx).read())

# 4. 创建 tree
tree_data = json.dumps({"base_tree": base_sha, "tree": [{"path": "SKILL.md", "mode": "100644", "type": "blob", "sha": blob['sha']}]}).encode()
req = urllib.request.Request(f"https://api.github.com/repos/{owner}/{repo['name']}/git/trees", data=tree_data, headers=headers, method='POST')
tree = json.loads(urllib.request.urlopen(req, timeout=10, context=ctx).read())

# 5. 创建 commit
commit_data = json.dumps({"message": "Initial commit", "tree": tree['sha'], "parents": [base_sha]}).encode()
req = urllib.request.Request(f"https://api.github.com/repos/{owner}/{repo['name']}/git/commits", data=commit_data, headers=headers, method='POST')

# 6. 更新 ref
ref_data = json.dumps({"sha": commit['sha'], "force": False}).encode()
req = urllib.request.Request(f"https://api.github.com/repos/{owner}/{repo['name']}/git/refs/heads/main", data=ref_data, headers=headers, method='PATCH')
```

**适用场景：** 只有 GitHub API 可达（api.github.com），但 gh CLI 未安装、github.com 网页被墙时。

**注意：** Token 需要 `repo` 权限，在 GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens 生成。

---

## 方案：Chromium 浏览器操作（Windows）

当需要用户登录 GitHub 生成 token，或者需要在知乎/B站等需要登录的平台搜索时，Chromium 浏览器（Chrome/Edge）的 `Chrome_WidgetWin_1` 窗口类**不接受后台键盘/鼠标事件**。

详细的 PowerShell SendKeys 操作参考见 `references/chromium-windows-automation.md`。

### 打开带 URL 的 Edge

```bash
# 先清理旧进程
taskkill /f /im msedge.exe 2>/dev/null

# 用 Python 打开新窗口到指定 URL
python -c "import subprocess; subprocess.Popen(['start', 'msedge', '--new-window', 'https://github.com/settings/tokens'], shell=True)"
```

### 设置地址栏 URL（可用）

```python
# set_value 可以设置地址栏内容（后台可用）
computer_use(action='set_value', element=ADDRESS_BAR_INDEX, value='https://github.com')
```

### 发送回车/键盘事件（需要 PowerShell）

Chromium 不接受后台 `key` / `type` 事件，必须用 PowerShell COM 自动化：

```powershell
# 激活 Edge 窗口
$wshell = New-Object -ComObject wscript.shell
$wshell.AppActivate('Edge')       # 窗口标题中包含的文本
Start-Sleep 1

# 发送按键
$wshell.SendKeys('{ENTER}')       # 回车
$wshell.SendKeys('^l')            # Ctrl+L（定位地址栏）
$wshell.SendKeys('^c')            # Ctrl+C
$wshell.SendKeys('^v')            # Ctrl+V
$wshell.SendKeys('{TAB}')         # Tab
```

在 git-bash 中调用：
```bash
powershell -Command "\$wshell = New-Object -ComObject wscript.shell; \$wshell.AppActivate('Edge'); Start-Sleep 1; \$wshell.SendKeys('{ENTER}')"
```

### 用 computer_use capture 读取页面

PowerShell 发送按键后，页面导航/交互完成，再用 `computer_use(action='capture', pid=PID, window_id=WINDOW_ID)` 读取页面的 accessibility tree 获取内容。

---

## 方案：国内网络下发现和评估代理服务

当暂无可用代理，需要购买机场时，可使用以下方法：

### 通过机场评测网站发现

部分机场评测/导航网站在国内可直接访问：
- `jichangtj.com` — 机场推荐与评测，含价格、线路、稳定性评估
- 使用 Python + urllib + 搜狗搜索进行发现和可用性测试

```python
# 测试多个机场官网的可达性
sites_to_test = [
    ('BoostNet', 'https://222.boostnet3.com'),
    ('贝贝云', 'https://222.2beibei.com'),
    ('悠兔', 'https://777.youtu6.shop'),
    ('TaiShan', 'https://jp.taishan.app'),
]
for name, url in sites_to_test:
    try:
        resp = urllib.request.urlopen(urllib.request.Request(url, headers=headers), timeout=6, context=ctx)
        print(f'✅ {name} 可达')
    except:
        print(f'❌ {name} 不可达')
```

### 典型价格参考

| 档次 | 价格 | 典型服务 |
|:----:|:----:|---------|
| 便宜 | ¥5-15/月 | 直连线路、少量冷门节点、够上 GitHub |
| 中等 | ¥15-30/月 | 中转线路、多节点、看视频 |
| 高配 | ¥30+/月 | IEPL专线、大带宽、流媒体解锁 |

### 鸡肉蛋问题处理

部分机场网站本身被墙（如 `ctc.run`），需要先有代理才能访问去买代理。解法：
1. 从可直接访问的机场买一个便宜的（如 BoostNet ¥15/月）
2. 用这个临时代理去访问被墙的网站买更好的
3. 或者用 URL 短链接（reurl.cc、bit.ly）绕过部分封锁

---

## 方案：拉取 raw 文件（配置、脚本等）

```bash
# 通过 ghproxy（推荐）
curl -sL "https://ghproxy.net/https://raw.githubusercontent.com/owner/repo/main/file"

# 用 Python（当 ghproxy 返回 403 时）
python3 -c "
import urllib.request, ssl
ctx = ssl._create_unverified_context()
r = urllib.request.urlopen('https://ghproxy.net/https://raw.githubusercontent.com/owner/repo/main/file', context=ctx)
open('file', 'wb').write(r.read())
"
```

## 镜像源列表 & 测试方法

参考 `references/china-mirrors.md` 获取最新的镜像可用性记录和完整测试命令。

## 国内网络下的工具安装镜像

部分开发工具（douyin-downloader、TouchDesigner 等）的官方源在国内受限。参考 `references/tool-install-mirrors.md` 获取各工具的 Gitee 镜像源、pip/npm 国内镜像、以及浏览器手动下载方案。

## 国内网络下的网页搜索

当 `web_search`、`web_extract`、`browser` 等标准 Hermes 搜索工具因网络/配置问题不可用时，参考 `references/china-web-search-foraging.md` —— 包含两种备选方案：

1. **浏览器 + 百度搜索**（推荐首选）—— 交互式，自带 AI 摘要，结果可点。通过 `browser_navigate` 直接访问百度搜索结果页，适合需要快速概览+深入阅读的场景。
2. **Python + 搜狗搜索**（备选）—— 可编程批量搜索，适合自动化场景，但可能触发反爬。

详见 `references/china-web-search-foraging.md` 中的完整流程和对比。

```bash
# 快速测试一个镜像是否可用
timeout 10 curl -sI "https://MIRROR/https://github.com/owner/repo" 2>&1 | grep -i "HTTP/"
```

## 已知限制

- **ghproxy.net** 对 git 协议的稳定性不如 HTTP，高频使用可能返回 502/403
- **镜像源会变化** — 被封后替换新的，定期更新 `references/china-mirrors.md`
- **`python3` 在 Windows 上可能是 `python`** — 注意别名差异
- **GitHub API 有频率限制**（未认证 60 req/h，认证后 5000 req/h）

### ⚠️ ghproxy SSL/TLS 握手失败 + git config insteadOf 陷阱

**现象：** `git clone` 报错 `schannel: failed to receive handshake, SSL/TLS connection failed`，URL 显示走的是 `mirror.ghproxy.com` 但实际用户已设置本地代理。

**根因：** 全局 git config 可能配置了 `url.https://mirror.ghproxy.com/https://github.com/.insteadof=https://github.com/`，导致所有 GitHub 操作自动走 ghproxy。当 ghproxy SSL 失效时，即使本地代理正常工作，git 也无法连接 GitHub。

**解法：** 临时取消 insteadOf 配置，用本地代理直连 clone，完成后恢复：

```bash
# 检查当前 insteadOf 配置
git config --global --get url.https://mirror.ghproxy.com/https://github.com/.insteadof
# 临时删除
git config --global --unset url.https://mirror.ghproxy.com/https://github.com/.insteadof
# 设置代理并克隆
export https_proxy=http://127.0.0.1:7897
git clone https://github.com/owner/repo.git
# 恢复配置
git config --global url.https://mirror.ghproxy.com/https://github.com/.insteadof https://github.com/
```
