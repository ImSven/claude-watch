<p align="center">
  <img src="logo.png" width="140" alt="Agent Watch Logo" />
</p>

<h1 align="center"><strong>Agent Watch</strong></h1>

<p align="center">
  Turn your iPhone into a remote Claude Code terminal.<br/>
  Monitor multiple sessions, approve permissions, send messages, and voice-control Claude — all from your phone.
</p>

---

```
                                    tmux send-keys
 iPhone  <=======>  WSL/Mac  <=====================>  Claude Code
  (App)    HTTP      Bridge        JSONL polling       (in tmux)
           SSE      (Node.js)     HTTP Hooks
```

## Features

- **Multi-session monitoring** — auto-discovers all Claude Code sessions running in tmux, displays each as a swipeable page
- **Live assistant text** — reads Claude's JSONL conversation files every 2 seconds, streams responses to your phone with markdown rendering
- **Tool activity feed** — shows Read, Edit, Write, Bash, Grep operations as they happen, with visual hierarchy (system ops are subtle, bash commands have code styling)
- **Permission approvals** — approve or deny Claude's actions from your phone (file edits, command execution, `AskUserQuestion` prompts with all options)
- **Remote command input** — type messages to any Claude session from your phone, injected via `tmux send-keys` for zero-latency delivery
- **Hold-to-speak voice input** — WeChat-style: toggle to mic mode, long press to record, release to send. Uses `SFSpeechRecognizer` with Chinese (zh-Hans) locale
- **Remembers connection** — saves the bridge IP so you don't re-enter it every time
- **Apple Watch support** — watchOS companion app with terminal output, permission prompts, and dictation input

## How It Works

### Input: Phone → Claude

Your phone sends a message → bridge finds the matching tmux pane by working directory → `tmux send-keys` injects the text directly into Claude's terminal input. Works whether Claude is idle or mid-turn.

### Output: Claude → Phone

1. Claude Code runs tools → HTTP hooks (`PostToolUse`, `PermissionRequest`, `Stop`) fire to the bridge
2. Bridge polls each session's JSONL conversation file every 2 seconds for new assistant text
3. All events stream to the phone via Server-Sent Events (SSE)

### Permission Flow

1. Claude hits a permission prompt → `PermissionRequest` hook **blocks** the response
2. Bridge pushes the prompt to phone with all options
3. User taps an option → bridge returns the decision → Claude continues

## Quick Start

### Prerequisites

- Linux (WSL) or macOS with Node.js 18+ and tmux
- Claude Code CLI installed
- iPhone on the same network

### 1. Install & start the bridge

```bash
cd skill/bridge
npm install
node server.js
```

```
╔═══════════════════════════════════════╗
║        AGENT WATCH BRIDGE             ║
╠═══════════════════════════════════════╣
║  Pairing Code:  648505                ║
║  IP Address:    172.20.8.218          ║
║  Port:          7860                  ║
║  Agents:        Claude                ║
╚═══════════════════════════════════════╝
```

### 2. Install Claude Code hooks

```bash
./skill/setup-hooks.sh
```

This adds HTTP hooks to `~/.claude/settings.json` so all Claude sessions stream events to the bridge.

### 3. Run Claude in tmux

```bash
tmux new-session -s dev
claude              # start Claude Code
```

The bridge auto-discovers Claude sessions in tmux panes when the phone connects.

### 4. Build the iOS app

```bash
cd ios/ClaudeWatch
xcodegen generate
open ClaudeWatch.xcodeproj
```

Set your Development Team, build and run on your iPhone.

### 5. Pair

1. Open the app → enter the bridge IP (remembered after first use)
2. Enter the 6-digit pairing code
3. All active Claude sessions appear as swipeable pages

## Architecture

```
claude-watch/
├── skill/
│   ├── bridge/
│   │   ├── server.js              # Bridge server (HTTP + SSE + Bonjour + tmux)
│   │   └── package.json
│   ├── setup-hooks.sh             # Install/remove Claude Code hooks
│   └── SKILL.md
│
├── ios/ClaudeWatch/
│   ├── Shared/                    # Shared iOS + watchOS
│   │   ├── Models/                # TerminalLine, AgentSession, WatchMessage, etc.
│   │   └── Extensions/            # Color+Hex, ClaudeMascot
│   │
│   ├── ClaudeWatch iOS/
│   │   ├── Views/
│   │   │   ├── PairingView.swift          # IP + 6-digit code entry
│   │   │   ├── ConnectionStatusView.swift # Multi-session pager, terminal, command input
│   │   │   └── SettingsView.swift
│   │   ├── Networking/
│   │   │   ├── BonjourDiscovery.swift
│   │   │   ├── BridgeClient.swift
│   │   │   └── SSEClient.swift
│   │   └── Services/
│   │       ├── RelayService.swift         # Bridge ↔ Watch coordinator
│   │       ├── SpeechService.swift        # SFSpeechRecognizer for voice input
│   │       └── NotificationService.swift
│   │
│   └── ClaudeWatch watchOS/       # Apple Watch companion
│       ├── Views/                 # Terminal, approval, voice input
│       └── Services/              # Watch-specific state + bridge client
```

## Bridge Server Details

### tmux Integration

The bridge scans `tmux list-panes` to discover all panes running `claude`. Each discovered session is exposed to the phone with its working directory as the identifier. Messages are injected via `tmux send-keys -t <target> '<message>' Enter`.

### JSONL Polling

Every 2 seconds, the bridge reads each session's Claude conversation file (`~/.claude/projects/<slug>/<session>.jsonl`) for new `assistant` entries. On first discovery, it looks back up to 8KB to catch recent responses.

### Hooks

| Hook | Purpose | Blocking? |
|------|---------|-----------|
| `PostToolUse` | Stream tool activity to phone | No |
| `PreToolUse` | Stream tool invocations | No |
| `PermissionRequest` | Forward permission prompts | **Yes** (up to 3 hours) |
| `Stop` | Detect turn completion | No |
| `Notification` | Idle/permission notifications | No |

### Phone Display

| Content | Style |
|---------|-------|
| Assistant text | Regular font, white, markdown rendered |
| User messages | Right-aligned orange bubble |
| Bash commands | Monospaced with dark background |
| System ops (Read/Edit/Write) | Small gray text with icon |
| Errors | Red with warning icon |

## Requirements

| Component | Version |
|-----------|---------|
| Node.js | 18+ |
| iOS | 17.0+ |
| watchOS | 10.0+ |
| Xcode | 16+ |
| Claude Code | 2.1+ |
| tmux | any |

## License

MIT
