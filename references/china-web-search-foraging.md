# China Web Search & Content Foraging

在中国大陆网络环境下，Hermes 标准工具（`web_search`、`web_extract`、`browser` Browserbase）可能因网络墙、代理限制或配置问题不可用。以下是从终端（git-bash）使用 Python + urllib + 国内搜索引擎的替代方案。

## 前提

- `python` 3.x 可用（`python3` 在 Windows git-bash 上通常不可用，用 `python`）
- `ssl._create_unverified_context()` 用于跳过 HTTPS 证书验证
- urllib 和 re 为 Python 标准库，无需额外安装

## 方案：通过搜狗搜索（Sogou）进行网页搜索

```python
import urllib.request, ssl, urllib.parse, re
ctx = ssl._create_unverified_context()
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}

query = urllib.parse.quote('你的中文搜索关键词')
url = f'https://www.sogou.com/web?query={query}'

req = urllib.request.Request(url, headers=headers)
resp = urllib.request.urlopen(req, timeout=15, context=ctx)
html = resp.read().decode('utf-8', errors='replace')

# 搜狗搜索结果解析：<h3> 标签内的 <a> 链接
results = re.findall(r'<h3[^>]*>.*?<a[^>]*href=\"([^\"]+)\"[^>]*>(.*?)</a>', html, re.DOTALL)
for link, title in results[:10]:
    t = re.sub(r'<[^>]+>', '', title).strip()
    t = re.sub(r'\s+', ' ', t)
    site = re.search(r'(?:https?://)?([^/]+)', link)
    print(f'📌 {t}')
    print(f'   [{site.group(1) if site else "?"}]')
```

### 命令行单行版

```bash
python -c "
import urllib.request, ssl, urllib.parse, re
ctx = ssl._create_unverified_context()
q = urllib.parse.quote('搜索关键词')
req = urllib.request.Request('https://www.sogou.com/web?query=' + q,
  headers={'User-Agent': 'Mozilla/5.0'})
html = urllib.request.urlopen(req, timeout=15, context=ctx).read().decode('utf-8','replace')
for l,t in re.findall(r'<h3[^>]*>.*?<a[^>]*href=\"([^\"]+)\"[^>]*>(.*?)</a>',html,re.DOTALL)[:5]:
  print(re.sub(r'<[^>]+>','',t).strip())
"
```

## 可用的国内搜索引擎

| 搜索引擎 | URL 模板 | 解析特点 | 实际可用性 (2025-2026) |
|----------|---------|---------|:-----------------:|
| **百度** | `https://www.baidu.com/s?wd=` | 自带 AI 摘要，结果丰富 | ✅ 浏览器搜索可用 |
| **搜狗 Sogou** | `https://www.sogou.com/web?query=` | `<h3>` 内 `<a>` 链接，较易解析 | ⚠️ 可能触发反爬 |
| **必应国内版** | `https://cn.bing.com/search?q=` | 结果包含 "二十大" 干扰 | ⚠️ 可能触发 Cloudflare |

### 推荐方案（根据场景）

| 场景 | 推荐 |
|------|------|
| 快速获取搜索结果概览 | **浏览器 + Baidu**（自带 AI 摘要，ref 链接可点击） |
| 纯命令行/自动批量搜索 | **Python + Sogou**（可编程解析） |
| 需要深入阅读具体文章 | **浏览器 + Baidu → 点击链接**（交互式浏览） |

## 方案：通过浏览器（browser_navigate）使用百度搜索

当标准 `web_search` 工具因后端配置不可用，或需要更丰富的搜索结果（AI摘要+可点击链接）时，使用 Hermes 浏览器工具直接访问百度搜索。

### 注意事项

1. **URL 编码**：浏览器 URL 含中文扩展字符时可能触发 `'utf-8' codec can't decode byte` 错误。始终对中文关键词进行百分号编码（`?wd=AI%E7%BB%98%E7%94%BB` 而非 `?wd=AI绘画`）。
2. **编码方式**：在 Python 中用 `urllib.parse.quote('中文')`，或在浏览器 URL 栏可以直接粘贴已编码的 URL 字符串。
3. **搜索结果页**包含：AI 摘要（顶部灰色框中）、精选笔记卡片、普通搜索结果链接（带 ref ID 可点击）。

### 流程

```python
# 1. 编码关键词
import urllib.parse
kw = urllib.parse.quote('AI绘画 取代 美术设计')

# 2. 通过 browser_navigate 打开百度搜索结果页
browser_navigate(url=f'https://www.baidu.com/s?wd={kw}')

# 3. 获取结果：browser_snapshot 返回含 ref 链接的树
#    百度页面自带 AI 摘要，快速获取关键结论

# 4. 点击感兴趣的结果标题（通过 ref ID）
browser_click(ref='e65')  # 需根据实际 snapshot 中 ref 取值

# 5. 读取文章内容
browser_snapshot()
```

### 与 Python+Sogou 方案的对比

| 维度 | 浏览器+Baidu | Python+Sogou |
|------|:----------:|:----------:|
| 反爬限制 | 低（浏览器模拟真人） | 高可能触发验证码 |
| AI 摘要 | ✅ 有 | ❌ 无 |
| 点击链接阅读 | ✅ 原生支持 | ❌ 需额外抓取 |
| 可编程/自动化 | ❌ 不能编程 | ✅ 完全可编程 |
| 适合场景 | 需要交互式浏览 | 批量/定时搜索 |

## 访问特定网站内容

以京东为例（提取商品标题和价格）：

```python
import urllib.request, ssl, re
ctx = ssl._create_unverified_context()
headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'}
req = urllib.request.Request('https://search.jd.com/Search?keyword=' + urllib.parse.quote('关键词'), headers=headers)
html = urllib.request.urlopen(req, timeout=15, context=ctx).read().decode('utf-8', 'replace')
# 根据网站结构自行解析
```

## 已知限制

1. **反爬机制**：很多网站（知乎、微信公众号、小红书）返回 403 或要求登录。需要用户帮登录后，用 Cookie 访问。
2. **JavaScript 渲染**：纯 urllib 无法执行 JS。B站的搜索结果、动态加载内容无法获取。
3. **速度**：每次请求约 1-5 秒，不适合高频轮询。
4. **编码**：Windows 终端输出中文可能乱码，用 `print()` 即可（Hermes 工具处理编码）。
5. **搜狗链接**：返回的是 `link?url=...` 格式的搜狗跳转链接，需要提取原始 URL。

## 何时使用此方案 vs 其他工具

| 场景 | 推荐 |
|------|------|
| 简单关键词搜索，获取标题+链接 | 此方案（Python+Sogou） |
| 需要看页面完整内容 | 此方案 + 再抓取具体页面，或让用户登录后用 `computer_use` 操作浏览器 |
| 需要动态 JS 内容/评论区 | 用户登录后，用 `computer_use` + `set_value` 填地址栏导航 |
| 需要交互式浏览（点击、翻页、填表单） | `computer_use` 桌面操作 |
| GitHub 文件/仓库相关 | `china-network-github` 技能中的 ghproxy 方案 |
