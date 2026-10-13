# Chromium 浏览器 Windows 自动化参考

## 问题描述

Chrome 和 Edge（均为 Chromium 内核）在 Windows 上使用 `Chrome_WidgetWin_1` 窗口类，该窗口类**不接受后台输入事件**。即通过 `computer_use` 的 `key` / `type` / `scroll` 等操作无法在后台工作。

## 可用操作

### ✅ 可以后台做的
- `computer_use(action='capture')` — 截图 + accessibility tree
- `computer_use(action='set_value', element=N, value='...')` — 设置地址栏/输入框内容（UIA ValuePattern）
- `computer_use(action='click', element=N)` — 点击按钮/链接
- `computer_use(action='focus_app', app='msedge')` — 聚焦窗口

### ❌ 不能后台做的
- `key` / `type` / `scroll` — 键盘/鼠标事件被 Chromium 拒绝

## PowerShell SendKeys 解决方案

通过 PowerShell COM 自动化绕过限制：

```powershell
$wshell = New-Object -ComObject wscript.shell
$wshell.AppActivate('Edge')       # 用窗口标题中的文本匹配
Start-Sleep 1                     # 等待窗口激活
$wshell.SendKeys('{ENTER}')       # 发送按键
```

### 常用按键

| 按键 | SendKeys 编码 |
|------|-------------|
| Enter | `{ENTER}` |
| Tab | `{TAB}` |
| Escape | `{ESC}` |
| Ctrl+A | `^a` |
| Ctrl+C | `^c` |
| Ctrl+V | `^v` |
| Ctrl+L | `^l` |
| F5 | `{F5}` |
| 方向键 | `{UP}` `{DOWN}` `{LEFT}` `{RIGHT}` |
| Alt+D | `%d` |

### AppActivate 匹配规则

`AppActivate(title)` 使用**子串匹配**：
- `AppActivate('知乎')` 匹配标题含"知乎"的窗口
- `AppActivate('Edge')` 匹配标题含"Edge"的窗口
- `AppActivate('首页')` 匹配标题含"首页"的窗口

返回 `True`（找到窗口）或 `False`（未找到）。

### 在 git-bash 中调用

```bash
# 注意 $ 需要转义
powershell -Command "\$wshell = New-Object -ComObject wscript.shell; \$wshell.AppActivate('Edge'); Start-Sleep 2; \$wshell.SendKeys('{ENTER}')"
```

### 完整工作流示例

```bash
# 1. 打开 Edge 到指定 URL
python -c "import subprocess; subprocess.Popen(['start', 'msedge', '--new-window', 'https://example.com'], shell=True)"

# 2. 等待页面加载
sleep 4

# 3. 用 set_value 设置地址栏（后台可行）
# computer_use(action='set_value', element=N, value='https://new-url.com')

# 4. 用 PowerShell 发送回车
powershell -Command "\$wshell = New-Object -ComObject wscript.shell; \$wshell.AppActivate('Edge'); Start-Sleep 1; \$wshell.SendKeys('{ENTER}')"

# 5. 等待页面加载
sleep 3

# 6. 用 computer_use capture 读取结果
```

## 注意事项

- `AppActivate` 会**暂时把窗口带到前台**，但执行成本低
- 多次 `SendKeys` 调用之间需要 `Start-Sleep` 间隔，太快可能丢失按键
- PowerShell `-Command` 中的 `$` 在 git-bash 中需要用 `\$` 转义
- 如果多个窗口匹配相同的标题，`AppActivate` 激活的是 Z-order 最高的那个
- 如果页面需要大量 JS 渲染（SPA），等待时间要更长
