# 国内网络下工具安装镜像记录

> 最后更新: 2026-07-22
> 网络环境: 中国大陆家庭宽带（无 VPN）

## 原则

当标准安装方式（GitHub clone、官网直连）被网络限制时，按以下优先级寻找替代源：

1. **Gitee 官方镜像** → `https://gitee.com/mirrors/<repo-name>` — 每日同步 GitHub 热门仓库
2. **npm/pip 国内镜像** → `registry.npmmirror.com` / `pypi.tuna.tsinghua.edu.cn` — CLI 工具和 Python 库的主力
3. **官方网站直接访问** → 某些被 GitHub 墙的官网（如 derivative.ca）在浏览器中直连可用
4. **winget/choco** → Windows 原生包管理器，不受 GitHub 封锁影响

## 各工具实测记录

### douyin-downloader (jiji262)

| 源 | 用途 | 实测 |
|---|------|:----:|
| `https://gitee.com/mirrors/douyin-downloader` | git clone | ✅ 可用，Gitee 官方镜像，每日同步 |
| `https://github.com/jiji262/douyin-downloader` | git clone | ❌ GitHub 被墙 |
| `pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple` | 依赖安装 | ✅ 可用 |

**安装步骤：**
```bash
git clone https://gitee.com/mirrors/douyin-downloader.git
cd douyin-downloader
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**Cookie 获取（必需的）：** 抖音 API 需要 Cookie 才能使用。用 Playwright：
```bash
pip install playwright
python -m playwright install chromium
python -m tools.cookie_fetcher --config config.yml
```
Playwright 会打开浏览器窗口 → 用户扫码登录 → 回到终端按 Enter → Cookie 自动写入 config.yml。

如果 cookie_fetcher 在 git-bash 后台模式下卡住（stdin 问题），可以用计算机视觉辅助：写一个自定义 launch_login.py 脚本，用 `asyncio.get_event_loop().run_in_executor(None, sys.stdin.readline)` 等待用户输入。

**关键 Cookie：** `ttwid` 是必需的，没有它 API 返回 Empty 200（反爬）。仅 `ttwid` 一个有效 Cookie 就足够下载公开视频。

### TouchDesigner

| 源 | 用途 | 实测 |
|---|------|:----:|
| `https://derivative.ca/download` | 下载页面 | ✅ 浏览器可访问 |
| `https://derivative.ca/download/release-download/38` | Windows 直链 | ❌ curl 返回 403 (需浏览器 session) |
| `https://derivative.ca/download/release-download/39` | macOS 直链 | ❌ 同上 |

**下载方式：** 必须通过浏览器手动操作，因为：
- Drupal colorbox modal 需要浏览器 session
- 安装包约 698MB（Web Installer）或 2.8GB（Full Installer）
- 下载前需注册 Derivative 账号

**非商业版（免费）：** 分辨率限制 1280×1280，功能完整，适合学习。

### Gitee 镜像搜索技巧

查询特定 GitHub 仓库是否有 Gitee 官方镜像：
```
搜索: site:gitee.com <owner> <repo-name> mirrors
```
Gitee 镜像命名模式: `https://gitee.com/mirrors/<repo-name>`
