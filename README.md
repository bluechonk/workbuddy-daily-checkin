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

| 操作 | playwright-cli 命令 | 说明 |
|---|---|---|
| 附加 | `attach --cdp=http://127.0.0.1:9222` | 接管已启动的 WorkBuddy |
| 打开账户菜单 | `click "[data-track-id=user_avatar_menu]"` | 头像；不要用 `find` / `eXXX` |
| 进入加油站 | `click "#fuel-menu-label"` | 菜单入口 |
| 查今日是否已领 | `eval "() => { const el=document.querySelector('#fuel-expanded-claim') \|\| [...document.querySelectorAll('*')].find(e=>e.id==='fuel-expanded-claim'); return el ? {text:el.textContent, disabled:el.disabled} : null }"` | 或见下方穿透版 |
| 点今日领取 | `click "#fuel-expanded-claim"` | 仅 `disabled=false` 时 |
| 认证入口 | `click "#fuel-action"` | 「认证领积分」 |
| 截图 | `screenshot --filename=<path>` | 分阶段落盘 |

完整选择器表：

| 元素 | selector | 备注 |
|---|---|---|
| 头像按钮 | `[data-track-id=user_avatar_menu]` | class 也可：`.user-menu-trigger`；`id` 形如 `:r1m:` 不稳定 |
| 菜单入口「Buddy加油站」 | `#fuel-menu-label` | |
| 「去邀约」 | `#fuel-menu-invite-label` | |
| 紧凑卡片 | `section.fuel-card.fuel-compact` | |
| 紧凑领取钮 | `#fuel-compact-claim` | 主界面头像旁 |
| 展开卡片 | `section.fuel-card.fuel-expanded` | |
| 活动标题 | `#fuel-title-expanded` | |
| 期次 | `#fuel-period` | |
| 今日领取 | `#fuel-expanded-claim` | 今日已领时 `disabled=true` |
| 认证领积分 | `#fuel-action` | |

> 快照 `e84` / `e101` 等 ref 每次 attach 都变，不要当记忆键。

Shadow DOM 内查 `disabled`（Playwright CSS 已能点，eval 需穿透）：

```bash
playwright-cli eval "() => {
  const visit = (root) => {
    if (!root || !root.querySelectorAll) return null;
    const el = root.querySelector('#fuel-expanded-claim');
    if (el) return el;
    for (const n of root.querySelectorAll('*')) {
      if (n.shadowRoot) {
        const r = visit(n.shadowRoot);
        if (r) return r;
      }
    }
    return null;
  };
  const el = visit(document);
  return el ? { text: el.textContent, disabled: el.disabled } : null;
}"
```

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
```

## 操作流水（完整可复制）

```powershell
# ── 启动（pwsh）────────────────────────────────────
$WorkBuddyExe = Get-WorkBuddyExe
$CDPPort = 9222
Get-Process WorkBuddy -ErrorAction SilentlyContinue | Stop-Process -Force
Start-Sleep -Seconds 2
Start-Process -FilePath $WorkBuddyExe -ArgumentList "--remote-debugging-port=$CDPPort"
Start-Sleep -Seconds 6
Invoke-WebRequest "http://127.0.0.1:$CDPPort/json/version" -UseBasicParsing
```

```bash
# ── 阶段1：附加 + 主界面截图 ───────────────────────
playwright-cli attach --cdp=http://127.0.0.1:9222
playwright-cli screenshot --filename=workbuddy-cdp.png

# ── 阶段2：打开账户菜单 + 截图 ─────────────────────
playwright-cli click "[data-track-id=user_avatar_menu]"
playwright-cli screenshot --filename=workbuddy-account-menu.png

# ── 阶段3：进入加油站 + 截图 ───────────────────────
playwright-cli click "#fuel-menu-label"
playwright-cli screenshot --filename=buddy-fuel-panel.png

# ── 状态：今日已领 / 认证（不要用 find 扫全文）────
playwright-cli eval "() => {
  const visit = (root) => {
    if (!root || !root.querySelectorAll) return null;
    const el = root.querySelector('#fuel-expanded-claim');
    if (el) return el;
    for (const n of root.querySelectorAll('*')) {
      if (n.shadowRoot) { const r = visit(n.shadowRoot); if (r) return r; }
    }
    return null;
  };
  const el = visit(document);
  return el ? { text: el.textContent, disabled: el.disabled } : null;
}"

# 仅当 disabled=false 时才领取：
# playwright-cli click "#fuel-expanded-claim"
```

## Shadow DOM

加油站在 shadow root 内（宿主如 `wb-slot--menu-signin`）。Playwright CSS 会穿透，`click "#fuel-menu-label"` 可直接用；裸 `document.querySelector` 打不进去。

## 实机结果（2026-09-14 复跑，三阶段）

**阶段 1 — CDP attach 后主界面**（无菜单）：

![阶段1 主界面](workbuddy-cdp.png)

**阶段 2 — 点头像后的账户菜单**（出现 Buddy加油站 入口）：

![阶段2 账户菜单](workbuddy-account-menu.png)

**阶段 3 — `click "#fuel-menu-label"` 进入加油站**（开学季 · 8期 · 今日已领 / 认证领积分）：

![阶段3 加油站面板](buddy-fuel-panel.png)

状态摘要：

- `#fuel-expanded-claim` =「今日已领」`disabled=true`（今日已领，未再点）
- `#fuel-action` =「认证领积分」可点（未点击，避免误触发）

## 限制

- 必须用 `--remote-debugging-port` 启动；普通启动连不上
- 应用重启后需重新 attach
- 端口默认 9222，改端口要同步改 attach URL
- `#fuel-*` / `data-track-id` 若改版失效，再用 snapshot + shadow 穿透重新定位
