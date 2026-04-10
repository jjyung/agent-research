# 🦞 OpenClaw — 個人 AI 助理

[![CI](https://github.com/openclaw/openclaw/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/openclaw/openclaw/actions/workflows/ci.yml?branch=main)
[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/openclaw/openclaw/blob/main/LICENSE)

OpenClaw 是一個部署在你自己裝置上的個人 AI 助理。它能在你已在使用的通訊頻道上回應你（WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、BlueBubbles、IRC、Microsoft Teams、Matrix、Feishu、LINE、Mattermost、Nextcloud Talk、Nostr、Synapse Chat、Tlon、Twitch、Zalo、WeChat、WebChat）。支援 macOS/iOS/Android 上的語音輸入與輸出，並可渲染一個你能控制的即時 Canvas。Gateway 只是 control plane，產品本身是那個助理。

- 官網：[openclaw.ai](https://openclaw.ai/)
- 文件：[docs.openclaw.ai](https://docs.openclaw.ai/)
- Discord：[discord.gg/clawd](https://discord.gg/clawd)

---

## 架構概覽

```
WhatsApp / Telegram / Slack / Discord / Google Chat / Signal / iMessage / ...
               │
               ▼
┌───────────────────────────────┐
│            Gateway            │
│       (control plane)         │
│     ws://127.0.0.1:18789      │
└──────────────┬────────────────┘
               │
               ├─ Pi agent (RPC)
               ├─ CLI (openclaw …)
               ├─ WebChat UI
               ├─ macOS app
               └─ iOS / Android nodes
```

核心理念：**Gateway as single control plane**。所有 channel、session、tool、event 都通過同一個 WebSocket 端點管理，實現統一的路由與控制。

---

## 主要功能亮點

### Local-first Gateway

- 單一 control plane，管理 sessions、channels、tools、events
- WebSocket 協議；預設綁定 loopback (`ws://127.0.0.1:18789`)
- 支援 Tailscale Serve/Funnel 作為安全的遠端存取方案

### Multi-channel Inbox

支援 20+ 種通訊平台，透過統一的 channel 抽象層接入

### Multi-agent Routing

可將不同 channel/account/peer 路由到獨立的 agent instance（各自的 workspace + session）

### Skills Platform

- Bundled skills（內建）、managed skills（透過 ClawHub 安裝）、workspace skills（自定義）
- 透過 `SKILL.md` 注入 agent prompt

### Voice Wake + Talk Mode

- macOS/iOS 支援喚醒詞
- Android 支援持續語音模式（ElevenLabs + 系統 TTS fallback）

### Live Canvas + A2UI

- Agent 驅動的視覺化 workspace
- 透過 A2UI 協議進行 push/reset/eval/snapshot

---

## 核心子系統

| 子系統            | 說明                                                           |
| ----------------- | -------------------------------------------------------------- |
| Gateway WebSocket | 單一 WS control plane，統管 clients、tools、events             |
| Pi agent runtime  | RPC 模式，支援 tool streaming 與 block streaming               |
| Session model     | `main` 處理直接對話；群組隔離；activation modes 與 queue modes |
| Browser control   | 專屬 Chrome/Chromium 實例，CDP 控制                            |
| Node system       | camera snap、screen record、location.get、notifications        |
| Skills            | SKILL.md 注入；ClawHub registry；workspace 自定義              |

---

## Agent-to-Agent (sessions\_\* tools)

支援跨 session 協作，不需切換通訊介面：

- `sessions_list` — 列出所有 active sessions 與 metadata
- `sessions_history` — 取得某 session 的對話記錄
- `sessions_send` — 對另一個 session 發送訊息；支援 reply-back ping-pong

---

## 安全模型

- 預設：主 session 工具可在 host 執行（完整存取）
- 群組/channel 安全：`agents.defaults.sandbox.mode: "non-main"` 讓非主 session 在 Docker sandbox 中執行
- DM pairing 機制：未知發送者會收到配對碼，bot 不處理其訊息，直到明確授權

---

## 技術棧

| 項目     | 內容                                                          |
| -------- | ------------------------------------------------------------- |
| 主要語言 | TypeScript (90%)、Swift (5.5%)、Kotlin (1.5%)                 |
| Runtime  | Node.js 24（建議）or 22.16+                                   |
| 套件管理 | pnpm（建議）                                                  |
| 測試框架 | Vitest（高度分片，多條 test lane）                            |
| 平台支援 | macOS app（Swift）、iOS node（Swift）、Android node（Kotlin） |

---

## 設計哲學

1. **Local-first**：Gateway 跑在使用者自己的裝置，資料不經第三方
2. **Channel as first-class citizen**：每個通訊平台都有完整的 adapter（allowlist、group routing、media caps）
3. **Extensibility via skills**：workspace skills 讓使用者自定義 agent 行為，而不是 fork 整個專案
4. **Agent workspace separation**：每個 agent 有獨立的 workspace root，透過 `AGENTS.md`、`SOUL.md`、`TOOLS.md` 客製化 identity 與行為
5. **Multi-agent by design**：session 模型從一開始就考慮多 agent 並行，可透過 `sessions_*` tools 組合協作
