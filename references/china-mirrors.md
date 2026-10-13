# GitHub 国内镜像源 & 代理可用性记录

> 最后更新: 2026-07-19
> 网络状态: 中国教育网/家庭宽带通用（VPN 未开启）

## 镜像源状态

| 镜像 | HTTP 文件下载 | git clone/fetch | API 调用 | 备注 |
|------|:---:|:---:|:---:|------|
| **`api.github.com`** (直连) | ✅ 可用 | N/A | ✅ 可用 | 最稳定，推荐读版本来 |
| **`ghproxy.net`** | ✅ 可用 (偶有 403) | ❌ 不稳定超时 | ❌ 不可用 | HTTP raw 下载可用 |
| **`github.com`** (直连) | ❌ 被墙 | ❌ 超时 | ❌ 被墙 | 完全不可用 |
| **`raw.githubusercontent.com`** | ❌ 被墙 | N/A | N/A | 需要代理 |
| **`gitclone.com`** | ❌ 不可用 | ❌ repo not found | ❌ 不可用 | 已失效 |
| **`github.moeyy.xyz`** | ❌ 无法连接 | ❌ 无法连接 | ❌ 无法连接 | DNS/连接失败 |
| **`mirror.ghproxy.com`** | ⏳ 待测试 | ⏳ 待测试 | ⏳ 待测试 | ghproxy 的 CNAME |

## 推荐策略（按优先级）

1. **读数据** → `api.github.com` 直连（版本号、release info、issue）
2. **下载 release zip** → `api.github.com` 的 zipball 端点
3. **下载 raw 文件** → `ghproxy.net/https://raw.githubusercontent.com/...` (HTTP)
4. **git clone/fetch** → `ghproxy.net/https://github.com/...` (可能超时，重试 2-3 次)
5. **更新已有仓库** → 先试 ghproxy remote，不行就 API 下载 zip 手动替换

## 测试命令

```bash
# 测试 ghproxy HTTP raw 文件
curl -sI --connect-timeout 10 "https://ghproxy.net/https://raw.githubusercontent.com/NousResearch/hermes-agent/main/README.md" 2>&1 | head -3

# 测试 ghproxy git clone
timeout 15 git clone --depth 1 "https://ghproxy.net/https://github.com/NousResearch/hermes-agent.git" /tmp/test-clone 2>&1

# 测试 GitHub API
curl -sL "https://api.github.com/repos/NousResearch/hermes-agent/releases/latest" -H "User-Agent: Mozilla/5.0"
```

## 文件获取替代方案

当所有镜像都不可用时：

### 通过 API 下载 release zip
```bash
curl -L -o repo.zip \
  -H "Authorization: token <GITHUB_TOKEN>" \
  "https://api.github.com/repos/owner/repo/zipball/v1.0.0"
```

### 通过 API 下载 release asset
```bash
# 1. 先获取 release ID
curl -sL "https://api.github.com/repos/owner/repo/releases/tags/v1.0.0"
# 2. 找到 asset 的 id 和 name
# 3. 下载（需要带 Accept header）
curl -L -o asset.zip \
  -H "Accept: application/octet-stream" \
  "https://api.github.com/repos/owner/repo/releases/assets/ASSET_ID"
```
