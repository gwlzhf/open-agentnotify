# agentnotify

**简体中文** | [English](README.en.md)

在 Claude Code、OMP、OpenCode CLI 或 Codex CLI 完成任务后，把通知推送到手机；也可以把权限审批交给手机。

## 便携版（无需安装）

从 Releases 下载对应平台的压缩包，解压即用（已附带 `cloudflared`）：

- Windows：解压后双击 `start.bat`。
- macOS：Apple 芯片选 `darwin-arm64`，Intel 选 `darwin-amd64`，解压后运行 `./start.sh`。
- Linux：解压后运行 `./start.sh`。

推送服务无法直连时，在程序旁的 `proxy.txt` 中写入代理地址，例如 `http://127.0.0.1:7890`。macOS 首次运行被拦截时，在解压目录执行 `xattr -dr com.apple.quarantine .`，或在“系统设置 → 隐私与安全性”中允许。

## 安装与启动

1. 安装 [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)（macOS 可用 `brew install cloudflared`）。
2. 安装 agentnotify：执行 `go install github.com/gwlzhf/agentnotify@latest`，或在仓库中执行 `go build -o agentnotify .`（Windows 为 `agentnotify.exe`）。确保可执行文件在 `PATH` 中。
3. 运行 `agentnotify start`，终端会显示二维码。用手机打开链接并点击“启用通知”。iOS 16.4 及以上需先“分享”→“添加到主屏幕”，再从主屏幕打开。
4. 添加其他设备：运行 `agentnotify pair`，或打开 http://localhost:9876/。

启动选项：`agentnotify start [--port 9876] [--reply-wait 0] [--no-tunnel] [--tunnel-url URL]`。

`--reply-wait N` 设置 Claude Code 和 Codex 完成通知后的手机回复等待时间（秒）：不传或为 `0` 时只通知；正数等待 N 秒（最多 600）；负数等待 600 秒。修改后需重启守护进程。OMP 与 OpenCode 的完成卡片始终可回复，不受此参数影响。

## 一键安装全局配置

```bash
agentnotify install              # 为已安装的客户端写入全局配置
agentnotify install claude omp   # 只处理指定客户端：claude | codex | workbuddy | opencode | omp
agentnotify uninstall            # 移除 agentnotify 写入的内容
```

未检测到的客户端会被跳过，确需写入时加 `--force`。移动可执行文件后重新运行 `install`。

## 审批与通知开关

```
agentnotify on       # 远程时开启，审批和提问转移到手机
agentnotify off      # 回到本机审批
agentnotify status
```

默认关闭，此时各客户端使用本机提示；完成通知不受影响。

## 发送通知与手机审批

```sh
agentnotify send "构建完成" "全部检查通过" --status ok      # --status: ok | fail | info
agentnotify ask "运行部署？" "生产环境" [--timeout 300] [--always]
agentnotify choose "用哪个数据库？" "说明" --option Redis --option SQLite [--multi] [--no-custom] [--timeout 300]
```

- `ask` 退出码：`0` 批准、`1` 拒绝、`2` 超时、`3` 守护进程未运行。`--always` 增加“始终允许”按钮，stdout 输出 `approve` 或 `always`。
- `choose` 支持单选、多选、自行输入和跳过。stdout 每行一个所选标签，`--json` 输出 JSON。退出码：`0` 已选、`1` 跳过、`2` 超时、`3` 守护进程未运行。
- 任意命令可用 `hooks/ask.sh "部署？" "生产环境" -- ./deploy.sh` 包装，审批通过才执行。

Android 和桌面系统可直接使用通知上的按钮；iOS 需点开通知后在应用内选择。

## 各客户端

### Claude Code

运行 `agentnotify install claude`。完成通知会在 Claude 结束时发送到手机，并可直接文字回复；Bash 等工具调用的审批及 `AskUserQuestion` 提问也会转到手机。项目级 `.claude/settings.json` 与全局配置二选一，避免重复触发。

### Codex CLI

运行 `agentnotify install codex`，然后在 Codex 的 `/hooks` 页面审阅并信任 hooks，重启 Codex。完成通知可回复，权限请求转到手机审批。

### OpenCode CLI

运行 `agentnotify install opencode`，或把 `.opencode/plugins/agentnotify.ts` 复制到项目或 `~/.config/opencode/plugins/`。重启 OpenCode 即自动加载。若 `agentnotify` 不在 `PATH` 中，先设置 `AGENTNOTIFY_BIN`：

```powershell
$env:AGENTNOTIFY_BIN = "C:\path\to\agentnotify.exe"
opencode
```

需要手机审批时，在 `opencode.json` 中开启权限询问：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "bash": "ask",
    "edit": "ask"
  }
}
```

### OMP

运行 `agentnotify install omp`。在本仓库中重启 OMP 会自动加载 `.omp/`。若 `agentnotify` 不在 `PATH` 中，先设置 `AGENTNOTIFY_BIN`。用 `omp --approval-mode write` 启动时，每次工具调用都会请求手机审批。

### WorkBuddy 桌面版

在可执行文件所在目录运行 `.\agentnotify.exe install workbuddy`，安装后重启 WorkBuddy 并审核启用 hooks。WorkBuddy 只发送通知，不接管审批；移除时运行 `agentnotify uninstall workbuddy`。

### 其他工程

`.claude/`、`.omp/`、`.opencode/` 和 `.codex/` 各自包含完整集成，可复制到其他工程；目标工程已有配置时请合并，不要直接覆盖。项目级与全局配置二选一，避免同一事件重复触发。

## 使用前提

各客户端都需要先运行 `agentnotify start` 并完成手机配对。
