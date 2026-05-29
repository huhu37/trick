# 在 WSL 下让 Agent CLI（Codex / Claude Code）结束后弹 Windows 通知

> 目标：在 WSL terminal 里跑 codex / claude 这类 CLI，**每次一轮对话/任务结束后**，让 Windows 弹出一条桌面通知。

---

## 核心原理

WSL 可以直接执行 Windows 的 `.exe`（通过 `/mnt/c/...` 访问 Windows 文件系统）。
所以思路统一为两层：

1. **触发点**：CLI 在「一轮结束」时执行一个 shell 脚本。
2. **弹窗脚本**：脚本里调用 Windows 的 `powershell.exe`，用 **BurntToast** 模块弹原生 Toast 通知。

两个 CLI 的弹窗脚本**完全一样**，区别只在「怎么触发」。

---

## 0. 前置依赖（只需做一次）

在 **Windows PowerShell** 里安装 BurntToast 模块：

```powershell
Install-Module -Name BurntToast -Scope CurrentUser -Force
```

---

## 1. 公共弹窗脚本

脚本内容（codex 用 `~/.codex/notify_on_finish.sh`，claude 用 `~/.claude/notify_on_finish.sh`，仅文案不同）：

```bash
#!/usr/bin/env bash

PS="/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe"

"$PS" -NoProfile -ExecutionPolicy Bypass -Command "
Import-Module BurntToast
New-BurntToastNotification -Text 'Claude Code', 'Claude is done.'
" >/dev/null 2>&1 &
```

| 部分 | 作用 |
|------|------|
| `/mnt/c/.../powershell.exe` | 从 WSL 调用 Windows PowerShell |
| `-NoProfile` | 不加载用户配置，启动更快 |
| `-ExecutionPolicy Bypass` | 临时绕过执行策略 |
| `Import-Module BurntToast` | 加载弹 Toast 的模块 |
| `New-BurntToastNotification -Text '标题','正文'` | 弹通知 |
| `>/dev/null 2>&1 &` | 丢弃输出 + 后台执行，不阻塞 CLI |

记得加可执行权限：`chmod +x ~/.codex/notify_on_finish.sh`（claude 同理）。

---

## 2. Codex 的接法

Codex 有内建配置项 `notify`，在 `~/.codex/config.toml` 里：

```toml
notify = ["/bin/bash", "-lc", "/home/xyzb/.codex/notify_on_finish.sh"]
```

- `notify` 的值是一个命令数组，codex 每完成一轮回合就执行它。
- 等价于运行 `/bin/bash -lc /home/xyzb/.codex/notify_on_finish.sh`。
- codex 还会把一个事件 JSON 作为参数追加传入，脚本里没用到，所以任何事件都会弹同样的通知。

> `config.toml` 里另有 `[tui] notifications = true`，那是 codex **终端内**的提示，和这套 Windows 弹窗是两回事。

---

## 3. Claude Code 的接法

Claude Code 没有 `notify` 配置项，但有 **Hooks** 机制。用 `Stop` 事件（每次 Claude 回答结束触发），配置在 `~/.claude/settings.json`：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/notify_on_finish.sh"
          }
        ]
      }
    ]
  }
}
```

- `Stop` hook 在 Claude 停止回答（含 clear / resume / compact）时触发。
- `type: "command"` 表示执行一条 shell 命令。
- 不需要 `matcher`（`Stop` 不针对特定工具）。

> 注意：把 hook 写进 settings.json 后，如果当前 session 启动时该目录没被监听，新 hook 可能不会立即生效。这时打开一次 `/hooks` 菜单（会重新加载配置）或重启 claude 即可。

---

## 4. 对照表

| | Codex | Claude Code |
|---|---|---|
| 触发机制 | `notify` 配置项 | `Stop` hook |
| 配置文件 | `~/.codex/config.toml` | `~/.claude/settings.json` |
| 弹窗脚本 | `~/.codex/notify_on_finish.sh` | `~/.claude/notify_on_finish.sh` |
| 弹窗实现 | powershell.exe + BurntToast | 同左（脚本一致） |
| 触发时机 | 每轮回合结束 | 每次回答结束（含 clear/compact 等） |

---

## 5. 验证

- **手动测试脚本**：`echo '{}' | bash ~/.claude/notify_on_finish.sh` —— 应立刻弹出通知。
- **测 agent**：跑任意一句让 agent 回答，结束后应弹通知。

