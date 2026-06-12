<p align="center">
  <img src="logo.png" width="140" alt="Agent Watch Logo" />
</p>

<h1 align="center"><strong>Agent Watch</strong></h1>

<p align="center">
  把你的 iPhone 变成远程 Claude Code 终端。<br/>
  同时监控多个会话、审批权限、发送消息、语音控制 Claude —— 全在手机上完成。
</p>

---

```
                                    tmux send-keys
 iPhone  <=======>  WSL/Mac  <=====================>  Claude Code
  (App)    HTTP      Bridge        JSONL 轮询          (tmux 中)
           SSE      (Node.js)     HTTP Hooks
```

## 功能特性

- **多会话监控** —— 自动发现 tmux 中运行的所有 Claude Code 会话，每个会话显示为可左右滑动的独立页面
- **实时回复推送** —— 每 2 秒轮询 Claude 的 JSONL 对话文件，将回复实时推送到手机，支持 Markdown 渲染
- **工具操作流** —— 实时显示 Read、Edit、Write、Bash、Grep 等操作，不同类型有清晰的视觉层次（系统操作小而淡，bash 命令有代码背景）
- **权限审批** —— 在手机上审批 Claude 的操作（文件编辑、命令执行、`AskUserQuestion` 问题及所有选项）
- **远程消息输入** —— 从手机向任意 Claude 会话发送消息，通过 `tmux send-keys` 注入，零延迟
- **按住说话** —— 微信风格：切换到语音模式后长按录音，松手自动发送。使用 `SFSpeechRecognizer`，默认中文识别
- **记住连接** —— 保存 Bridge IP 地址，无需每次重新输入
- **Apple Watch 支持** —— watchOS 配套应用，支持终端输出、权限审批和语音输入

## 工作原理

### 输入：手机 → Claude

手机发送消息 → Bridge 根据工作目录找到对应的 tmux 窗格 → `tmux send-keys` 将文本直接注入 Claude 的终端输入。无论 Claude 是空闲还是正在工作都能即时送达。

### 输出：Claude → 手机

1. Claude Code 执行工具 → HTTP Hooks（`PostToolUse`、`PermissionRequest`、`Stop`）将事件发送到 Bridge
2. Bridge 每 2 秒轮询每个会话的 JSONL 对话文件，读取新的 assistant 回复
3. 所有事件通过 Server-Sent Events (SSE) 实时推送到手机

### 权限审批流程

1. Claude 遇到权限提示 → `PermissionRequest` Hook **阻塞**等待响应
2. Bridge 将提示推送到手机，显示所有选项
3. 用户点击选项 → Bridge 将决定返回给 Claude → Claude 继续或停止

## 快速开始

### 前置条件

- Linux (WSL) 或 macOS，已安装 Node.js 18+ 和 tmux
- 已安装 Claude Code CLI
- iPhone 与主机在同一网络

### 1. 安装并启动 Bridge

```bash
cd skill/bridge
npm install
node server.js
```

```
╔═══════════════════════════════════════╗
║        AGENT WATCH BRIDGE             ║
╠═══════════════════════════════════════╣
║  配对码:      648505                  ║
║  IP 地址:     172.20.8.218            ║
║  端口:        7860                    ║
║  Agents:      Claude                  ║
╚═══════════════════════════════════════╝
```

### 2. 安装 Claude Code Hooks

```bash
./skill/setup-hooks.sh
```

这会在 `~/.claude/settings.json` 中添加 HTTP Hooks，让所有 Claude 会话的事件自动发送到 Bridge。

### 3. 在 tmux 中启动 Claude

```bash
tmux new-session -s dev
claude              # 启动 Claude Code
```

手机连接时，Bridge 会自动发现 tmux 窗格中的所有 Claude 会话。

### 4. 编译 iOS 应用

```bash
cd ios/ClaudeWatch
xcodegen generate
open ClaudeWatch.xcodeproj
```

在 Xcode 中设置开发者团队，编译并安装到 iPhone。

### 5. 配对

1. 打开 App → 输入 Bridge IP（首次输入后自动记住）
2. 输入 6 位配对码
3. 所有活跃的 Claude 会话自动显示为可滑动页面

## 项目结构

```
claude-watch/
├── skill/
│   ├── bridge/
│   │   ├── server.js              # Bridge 服务器（HTTP + SSE + Bonjour + tmux）
│   │   └── package.json
│   ├── setup-hooks.sh             # 安装/卸载 Claude Code Hooks
│   └── SKILL.md
│
├── ios/ClaudeWatch/
│   ├── Shared/                    # iOS + watchOS 共享代码
│   │   ├── Models/                # TerminalLine, AgentSession, WatchMessage 等
│   │   └── Extensions/            # Color+Hex, ClaudeMascot
│   │
│   ├── ClaudeWatch iOS/
│   │   ├── Views/
│   │   │   ├── PairingView.swift          # IP + 配对码输入
│   │   │   ├── ConnectionStatusView.swift # 多会话页面、终端、消息输入
│   │   │   └── SettingsView.swift
│   │   ├── Networking/
│   │   │   ├── BonjourDiscovery.swift
│   │   │   ├── BridgeClient.swift
│   │   │   └── SSEClient.swift
│   │   └── Services/
│   │       ├── RelayService.swift         # Bridge ↔ Watch 协调器
│   │       ├── SpeechService.swift        # 语音识别（SFSpeechRecognizer）
│   │       └── NotificationService.swift
│   │
│   └── ClaudeWatch watchOS/       # Apple Watch 配套应用
│       ├── Views/                 # 终端、审批、语音输入
│       └── Services/              # Watch 专用状态 + Bridge 客户端
```

## Bridge 服务器详情

### tmux 集成

Bridge 通过 `tmux list-panes` 扫描所有运行 `claude` 的窗格。每个发现的会话以工作目录作为标识暴露给手机。消息通过 `tmux send-keys -t <target> '<message>' Enter` 注入。

### JSONL 轮询

每 2 秒读取每个会话的 Claude 对话文件（`~/.claude/projects/<slug>/<session>.jsonl`），提取新的 `assistant` 条目。首次发现时回溯最多 8KB 以捕获最近的回复。

### Hooks 列表

| Hook | 用途 | 是否阻塞 |
|------|------|----------|
| `PostToolUse` | 推送工具操作到手机 | 否 |
| `PreToolUse` | 推送工具调用 | 否 |
| `PermissionRequest` | 转发权限提示 | **是**（最长 3 小时）|
| `Stop` | 检测回合结束 | 否 |
| `Notification` | 空闲/权限通知 | 否 |

### 手机端显示样式

| 内容 | 样式 |
|------|------|
| Assistant 回复 | 常规字体、白色、支持 Markdown |
| 用户消息 | 右对齐橙色气泡 |
| Bash 命令 | 等宽字体、深色背景 |
| 系统操作（Read/Edit/Write）| 小号灰色文字 + 图标 |
| 错误 | 红色 + 警告图标 |

## 环境要求

| 组件 | 版本 |
|------|------|
| Node.js | 18+ |
| iOS | 17.0+ |
| watchOS | 10.0+ |
| Xcode | 16+ |
| Claude Code | 2.1+ |
| tmux | 任意版本 |

## 许可证

MIT
