# WorkBuddy 每日签到 — CDP DOM 操控

## 设想

WorkBuddy 是 Electron（Chromium）套壳。启动时加 `--remote-debugging-port`，即可用 Playwright 通过 CDP 接管其渲染进程，直接读/点 DOM 完成签到。

## 环境（2026-09-14 实测）

| 项 | 值 |
|---|---|
| 应用 | WorkBuddy 5.5.6 |
| 运行时 | Electron 37.10.3 / Chrome 138.0.7204.251 |
| 工具 | `playwright-cli` 0.1.19（`@playwright/cli`，npm 全局） |
| CDP | `http://127.0.0.1:9222` |

结论：**方案可行**，已 attach 成功并读到完整 DOM。

## 定位 WorkBuddy.exe（pwsh 动态路径）

不要写死用户目录。优先查已知安装位置，找不到再按 `Program Files` / `LocalAppData\Programs` 兜底搜索。

```powershell
function Get-WorkBuddyExe {
    $candidates = @(
        (Join-Path $env:LOCALAPPDATA 'Programs\WorkBuddy\WorkBuddy.exe')
        (Join-Path $env:ProgramFiles 'WorkBuddy\WorkBuddy.exe')
        (Join-Path ${env:ProgramFiles(x86)} 'WorkBuddy\WorkBuddy.exe')
    )
    foreach ($p in $candidates) {
        if ($p -and (Test-Path -LiteralPath $p)) { return (Resolve-Path -LiteralPath $p).Path }
    }

    $searchRoots = @(
        (Join-Path $env:ProgramFiles '*')
        (Join-Path ${env:ProgramFiles(x86)} '*')
        (Join-Path $env:LOCALAPPDATA 'Programs\*')
    ) | Where-Object { $_ -and (Test-Path -LiteralPath (Split-Path $_ -Parent)) }

    foreach ($root in $searchRoots) {
        $hit = Get-ChildItem -Path $root -Filter 'WorkBuddy.exe' -Recurse -ErrorAction SilentlyContinue |
            Select-Object -First 1
        if ($hit) { return $hit.FullName }
    }

    throw 'WorkBuddy.exe not found. Install WorkBuddy or pass -WorkBuddyExe explicitly.'
}

$WorkBuddyExe = Get-WorkBuddyExe
Write-Host "WorkBuddy: $WorkBuddyExe"
```

## 操作流水

```powershell
# 0. 解析 exe 路径（见上）
$WorkBuddyExe = Get-WorkBuddyExe
$CDPPort = 9222

# 1. 杀掉已有实例（默认启动没有调试端口）
Get-Process WorkBuddy -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 2

# 2. 带调试端口重启
Start-Process -FilePath $WorkBuddyExe -ArgumentList "--remote-debugging-port=$CDPPort"
Start-Sleep -Seconds 6

# 3. 确认 CDP
Invoke-WebRequest "http://127.0.0.1:$CDPPort/json/version" -UseBasicParsing
```

```bash
# 4. 附加（attach，不是 open）
playwright-cli attach --cdp=http://127.0.0.1:9222

# 5. 打开用户菜单（头像按钮 ref 以当次 snapshot 为准）
playwright-cli find "セシリア"
playwright-cli click e84

# 6. 精准进入 Buddy加油站
playwright-cli click "#fuel-menu-label"

# 7. 校验
playwright-cli find "今日已领"
playwright-cli find "认证领积分"
playwright-cli screenshot
```

## Shadow DOM 注意点

加油站 UI 在 shadow root 内，宿主例如：

- `div.wb-slot.wb-slot--menu-signin`
- `div.wb-slot--closable-content`
- `div.wb-slot.wb-slot--menu-growth-content`

`document.querySelector('#fuel-...')` 默认打不进去。Playwright 的 CSS 选择器会自动穿透，可直接 `click "#fuel-menu-label"`。若用 `eval`，需自行遍历 `el.shadowRoot`。

## 选择器

| 元素 | id | class |
|---|---|---|
| 菜单入口「Buddy加油站」 | `#fuel-menu-label` | `fuel-menu-entry__label` |
| 「去邀约」 | `#fuel-menu-invite-label` | `fuel-menu-entry__label` |
| 紧凑卡片 | — | `section.fuel-card.fuel-compact` |
| 紧凑领取钮 | `#fuel-compact-claim` | `fuel-btn` |
| 展开卡片 | — | `section.fuel-card.fuel-expanded` |
| 活动标题 | `#fuel-title-expanded` | `fuel-expanded-title` |
| 期次 | `#fuel-period` | `fuel-period` |
| 今日领取 | `#fuel-expanded-claim` | `fuel-btn` |
| 认证领积分 | `#fuel-action` | `fuel-btn fuel-secondary` |

## 当前账号状态（2026-09-14）

```
开学季 / Buddy加油站·8期 / 9/15结束
今日可领100积分 · 已领6天 · 累计600分
#fuel-expanded-claim  今日已领   disabled=true
#fuel-action          认证领积分 disabled=false
```

本次未执行领取（今日已领），未点击「认证领积分」。

## 限制

- 必须用 `--remote-debugging-port` 启动；普通启动连不上
- 应用重启后需重新 attach
- 端口默认 9222，改端口要同步改 attach URL
