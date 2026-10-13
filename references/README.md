# GitHub 国内网络操作指南 - 综述

本 skill 包含以下操作方案：

## 核心方案（SKILL.md 正文）

| 方案 | 适用场景 |
|------|---------|
| git clone 通过 ghproxy | GitHub 网页被墙但 git 可用时 |
| GitHub API 创建仓库并推送 | 只有 api.github.com 可达时 |
| Chromium 浏览器操作 | 需要用户登录 GitHub/web 平台时 |
| 国内网络下代理服务发现 | 需要先购买代理才能上 GitHub 时 |
| 拉取 raw 文件 | 需要下载配置文件/脚本时 |
| 检查本地仓库是否为最新 | 无法 fetch 但需要确认版本时 |
| 国内网络网页搜索 | 标准搜索工具不可用时的替代方案 |

## 参考文件

| 文件 | 内容 |
|------|------|
| `references/china-mirrors.md` | 镜像源可用性测试和列表 |
| `references/china-web-search-foraging.md` | 使用搜狗等国内搜索引擎进行网页搜索 |
| `references/chromium-windows-automation.md` | PowerShell SendKeys 绕过 Chromium 输入限制 |

## 关键限制

- GitHub API 未认证 60 req/h，认证后 5000 req/h
- ghproxy.net 高频使用可能返回 502/403
- python3 在 Windows 上是 python（注意别名差异）
- Chromium `Chrome_WidgetWin_1` 不接受后台键盘事件
- 部分机场（代理服务）官网也在墙内，形成"需要代理买代理"死循环
