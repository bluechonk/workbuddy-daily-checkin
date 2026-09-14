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

结论：**方案可行**。`#fuel-*` 选择器已实测稳定，日常操作直接打 id，不必每次全页 `find`。

## 稳定选择器（实操已确认，直接用）

| 操作 | 命令 | 说明 |
|---|---|---|
| 进入加油站 | `playwright-cli click "#fuel-menu-label"` | 菜单入口，id 稳定 |
| 读今日领取态 | `playwright-cli find "今日已领"` 或看 `#fuel-expanded-claim` | `disabled=true` 表示今日已领 |
| 认证入口 | `playwright-cli click "#fuel-action"` | 「认证领积分」，id 稳定 |

完整 id 表：

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

> 快照里的 `e84` / `e101` 等 **ref 每次 attach 都会变**，不能当长期记忆键。头像按钮无固定 id，需要时再 `find "セシリア"`。

## 定位 WorkBuddy.exe（pwsh 动态路径）

```powershell
function Get-WorkBuddyExe {
    $roots = @(
        $env:LOCALAPPDATA
        $env:ProgramFiles
        ${env:ProgramFiles(x86)}
    ) | Where-Object { $_ }

    foreach ($r in $roots) {
        $p = if ($r -eq $env:LOCALAPPDATA) {
            Join-Path $r 'Programs\WorkBuddy\WorkBuddy.exe'
        } else {
            Join-Path $r 'WorkBuddy\WorkBuddy.exe'
        }
        if (Test-Path -LiteralPath $p) { return (Resolve-Path -LiteralPath $p).Path }
    }

    foreach ($r in $roots) {
        $prefix = if ($r -eq $env:LOCALAPPDATA) { Join-Path $r 'Programs' } else { $r }
        if (-not (Test-Path -LiteralPath $prefix)) { continue }
        $hit = Get-ChildItem -Path $prefix -Filter 'WorkBuddy.exe' -Recurse -ErrorAction SilentlyContinue |
            Select-Object -First 1
        if ($hit) { return $hit.FullName }
    }

    throw 'WorkBuddy.exe not found.'
}

$WorkBuddyExe = Get-WorkBuddyExe
$CDPPort = 9222
```

## 操作流水

```powershell
$WorkBuddyExe = Get-WorkBuddyExe
$CDPPort = 9222

Get-Process WorkBuddy -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 2
Start-Process -FilePath $WorkBuddyExe -ArgumentList "--remote-debugging-port=$CDPPort"
Start-Sleep -Seconds 6
Invoke-WebRequest "http://127.0.0.1:$CDPPort/json/version" -UseBasicParsing
```

```bash
playwright-cli attach --cdp=http://127.0.0.1:9222

# 需要先打开账户菜单时（头像无固定 id）
playwright-cli find "セシリア"
playwright-cli click <当次ref>

# 之后固定走 id，不必重扫
playwright-cli click "#fuel-menu-label"
playwright-cli find "今日已领"
playwright-cli find "认证领积分"
playwright-cli screenshot
```

## Shadow DOM

加油站在 shadow root 内（宿主如 `wb-slot--menu-signin`）。Playwright CSS 会穿透，`click "#fuel-menu-label"` 可直接用；裸 `document.querySelector` 打不进去。

## 实机结果（2026-09-14 复跑）

CDP 附加后主界面：

![WorkBuddy CDP 主界面](workbuddy-cdp.png)

进入 Buddy加油站后（开学季 · 8期 · 今日已领 / 认证领积分）：

![Buddy加油站面板](buddy-fuel-panel.png)

按 README 流水复跑校验：

![复跑校验](run-readme-verify.png)

状态摘要：

- `#fuel-expanded-claim` =「今日已领」`disabled=true`（今日已领，未再点）
- `#fuel-action` =「认证领积分」可点（未点击，避免误触发）

## 限制

- 必须用 `--remote-debugging-port` 启动；普通启动连不上
- 应用重启后需重新 attach
- 端口默认 9222，改端口要同步改 attach URL
- `#fuel-*` 若改版失效，再用 `find` / shadow 穿透重新定位
