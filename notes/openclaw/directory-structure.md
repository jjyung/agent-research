# 目錄結構分析

> 分析日期：2026-04-10  
> 來源：[sources/openclaw](../../sources/openclaw)

---

## 頂層目錄總覽

```
openclaw/
├── src/          # 核心業務邏輯（TypeScript）
├── extensions/   # 所有 channel adapter 與 model provider plugin
├── packages/     # 內部共享 package（SDK、plugin contract）
├── apps/         # 原生平台 app（macOS/iOS/Android）
├── ui/           # Web Control UI（React/Vite）
├── skills/       # 內建 skill 定義（SKILL.md 集合）
├── Swabble/      # macOS Swift framework（protocol library）
├── docs/         # 官方文件原始檔
├── qa/           # QA 評測場景與 harness
├── scripts/      # 建置、CI、維護腳本
├── test/         # 整合測試與 e2e fixtures
├── test-fixtures/# 測試用的靜態資料
├── vendor/       # vendored 第三方 library（a2ui）
└── .agents/      # agent maintainer 指引（AI-readable）
```

---

## `src/` — 核心業務邏輯

TypeScript monolith，包含 Gateway 的所有核心功能。檔案以功能前綴命名（扁平結構），搭配少量子目錄。

### 主要子模組

| 目錄/前綴 | 功能 |
|-----------|------|
| `src/gateway/` | **Gateway server 主體**：WebSocket control plane、HTTP endpoints、auth、session 管理、config reload、Tailscale 整合、MCP HTTP server |
| `src/agents/` | **Agent runtime 核心**：pi-embedded runner（LLM 呼叫）、工具執行策略、subagent registry、bash-tools、skills 載入、model 選擇與 failover、system prompt 組合 |
| `src/channels/` | **Channel 抽象層**：allowlist、mention gating、typing indicator、reply prefix、session 與 channel 的綁定邏輯；具體 channel 在 `extensions/` |
| `src/sessions/` | **Session 模型**：session ID、lifecycle events、transcript events、send policy、model override |
| `src/acp/` | **ACP（Agent Communication Protocol）**：agent-to-agent 通訊協議 binding |
| `src/auto-reply/` | 自動回覆核心邏輯 |
| `src/hooks/` | Webhook 觸發機制 |
| `src/cron/` | Cron job 管理 |
| `src/mcp/` | MCP（Model Context Protocol）transport 與 config |
| `src/tui/` | 終端機 UI（TUI）|
| `src/web/` | WebChat frontend 服務 |
| `src/wizard/` | Onboarding wizard 流程 |
| `src/memory-host-sdk/` | Memory host SDK（also duplicated as package） |
| `src/process/` | Process 管理 |
| `src/media/` | 媒體檔案 pipeline（圖片/音訊/影片） |
| `src/shared/` | 跨模組共享工具 |
| `src/scripts/` | 內部腳本（build-time） |

### 命名規範觀察

- 單一檔案以 `<domain>-<concern>.ts` 命名（例：`session-transcript-key.ts`、`bash-tools.exec.ts`）
- 測試與實作並排：`foo.ts` / `foo.test.ts`
- Runtime 版本（帶 side effect）加 `.runtime.ts`
- E2E 測試加 `.e2e.test.ts`；live 測試（需外部依賴）加 `.live.test.ts`

---

## `extensions/` — Channel 與 Model Provider Plugins

約 80+ 個子目錄，每個對應一個 channel adapter 或 model provider。這是 OpenClaw 支援多平台的核心機制。

### Channel Adapters（通訊平台）

| 目錄 | 平台 |
|------|------|
| `whatsapp/` | WhatsApp（Baileys） |
| `telegram/` | Telegram（grammY） |
| `discord/` | Discord（discord.js） |
| `slack/` | Slack（Bolt） |
| `googlechat/` | Google Chat |
| `signal/` | Signal（signal-cli） |
| `bluebubbles/` | iMessage via BlueBubbles |
| `imessage/` | iMessage（legacy） |
| `msteams/` | Microsoft Teams |
| `matrix/` | Matrix |
| `feishu/` | 飛書 |
| `irc/` | IRC |
| `line/` | LINE |
| `mattermost/` | Mattermost |
| `nextcloud-talk/` | Nextcloud Talk |
| `nostr/` | Nostr |
| `synology-chat/` | Synology Chat |
| `tlon/` | Tlon |
| `twitch/` | Twitch |
| `zalo/` + `zalouser/` | Zalo |
| `qqbot/` | QQ Bot |
| `xiaomi/` | 小米 |

### Model Providers（LLM 接入）

| 目錄 | Provider |
|------|----------|
| `anthropic/` | Anthropic（Claude） |
| `openai/` | OpenAI |
| `google/` | Google（Gemini） |
| `microsoft/` | Azure OpenAI |
| `amazon-bedrock/` | AWS Bedrock |
| `deepseek/` | DeepSeek |
| `ollama/` | Ollama（本地） |
| `groq/` | Groq |
| `mistral/` | Mistral |
| `xai/` | xAI（Grok） |
| `openrouter/` | OpenRouter |
| `litellm/` | LiteLLM proxy |
| `vercel-ai-gateway/` | Vercel AI Gateway |
| `cloudflare-ai-gateway/` | Cloudflare AI Gateway |
| `github-copilot/` | GitHub Copilot |
| `kilocode/` + `kimi-coding/` | 程式碼特化 LLM |

### 其他功能性 Extension

| 目錄 | 功能 |
|------|------|
| `browser/` | Browser control（CDP） |
| `memory-core/` + `memory-lancedb/` + `memory-wiki/` | 記憶系統 |
| `active-memory/` | 即時記憶 |
| `diffs/` | Diff 顯示工具 |
| `elevenlabs/` | TTS（ElevenLabs） |
| `deepgram/` | 語音轉文字 |
| `voice-call/` | 語音通話 |
| `talk-voice/` | Talk mode（語音對話） |
| `acpx/` | ACP 擴充 |
| `device-pair/` | 裝置配對 |
| `webhooks/` | Webhook 觸發 |
| `image-generation-core/` | 圖片生成核心 |
| `video-generation-core/` | 影片生成核心 |
| `shared/` | Extension 間共享模組 |

---

## `packages/` — 內部共享 Packages

| 目錄 | 功能 |
|------|------|
| `plugin-sdk/` | **Plugin SDK**：external plugin 開發介面，定義 plugin 的 hook、tool、型別 |
| `plugin-package-contract/` | Plugin 的 package.json schema 與合約驗證 |
| `memory-host-sdk/` | Memory host 的 SDK（供 memory extension 用） |
| `clawdbot/` | Bot framework 抽象層 |
| `moltbot/` | Moltbot（lobster bot，OpenClaw 的 mascot bot） |

---

## `apps/` — 原生平台 App

| 目錄 | 技術 | 功能 |
|------|------|------|
| `macos/` | Swift | macOS menu bar app；Voice Wake/PTT；WebChat；remote gateway control |
| `ios/` | Swift | iOS node；Canvas；Voice Wake；Talk Mode；camera |
| `android/` | Kotlin | Android node；Chat/Voice/Canvas tabs；device commands |
| `shared/` | Swift/Kotlin 共享邏輯 | 跨平台共享的 protocol 型別 |

---

## `ui/` — Web Control UI

React + Vite 的前端，直接由 Gateway HTTP server 提供服務。

- `src/` — React 元件與業務邏輯
- `public/` — 靜態資源
- `vite.config.ts` — 打包設定
- `vitest.config.ts` — 前端單元測試
- `vitest.node.config.ts` — Node 環境測試

---

## `skills/` — 內建 Skill 定義

約 50+ 個子目錄，每個是一個可安裝的 skill，包含 `SKILL.md`（注入 agent 的 prompt 指引）及相關工具腳本。

### 精選 Skills

| 目錄 | 功能 |
|------|------|
| `taskflow/` | 任務管理工作流 |
| `coding-agent/` | 程式碼 agent |
| `github/` | GitHub 操作 |
| `notion/` + `obsidian/` + `bear-notes/` | 筆記工具整合 |
| `discord/` + `slack/` | 直接操作對應平台 |
| `1password/` | 密碼管理 |
| `weather/` | 天氣查詢 |
| `spotify-player/` | Spotify 控制 |
| `peekaboo/` | macOS 截圖工具 |
| `skill-creator/` | 建立新 skill 的 meta-skill |
| `clawhub/` | ClawHub registry 整合 |
| `tmux/` | tmux 控制 |
| `summarize/` | 摘要工具 |
| `canvas/` | Canvas 操作 |
| `oracle/` | 自定義知識庫查詢 |

---

## `Swabble/` — macOS Swift Framework

獨立的 Swift Package，提供 OpenClaw Gateway 協議的 Swift binding。iOS 和 macOS app 透過此 framework 與 Gateway 通訊。

```
Swabble/
├── Sources/   # Swift 原始碼（protocol types、WebSocket client）
├── Tests/     # Swift 測試
└── docs/      # API 文件
```

---

## `qa/` — QA 評測場景

AI behavior 評測框架，用來驗證 agent 行為符合預期。

| 目錄/檔案 | 功能 |
|-----------|------|
| `scenarios/` | 評測場景定義 |
| `scenarios.md` | 場景清單 |
| `frontier-harness-plan.md` | Frontier model 測試計畫 |
| `new-scenarios-2026-04.md` | 新增場景（2026/04） |

---

## `vendor/` — Vendored 第三方 Library

| 目錄 | 說明 |
|------|------|
| `a2ui/` | A2UI framework（agent 驅動的 Canvas UI protocol），由 Mario Zechner 開發，vendored 在此 |

---

## `.agents/` — Agent 維護指引

供 AI coding assistant（如 Claude）閱讀的維護指引，定義如何協助維護此 codebase：commit 規範、PR 原則、測試要求、架構邊界等。

---

## Vitest 測試架構觀察

根目錄有 **50+ 個 `vitest.*.config.ts`**，代表高度分片的測試策略：

| 分片類型 | 說明 |
|----------|------|
| `vitest.unit.config.ts` | 純單元測試 |
| `vitest.unit-fast.config.ts` | 快速單元測試（CI 加速） |
| `vitest.e2e.config.ts` | End-to-end 測試 |
| `vitest.live.config.ts` | 需要真實 API key 的 live 測試 |
| `vitest.extension-<name>.config.ts` | 各 extension 獨立測試 lane |
| `vitest.full-*.config.ts` | 完整測試套件（分 shard 執行） |
| `vitest.gateway.config.ts` | Gateway 整合測試 |
| `vitest.agents.config.ts` | Agent 行為測試 |

**設計哲學**：測試按相依性與執行時間分層，CI 可並行執行多條 lane，避免單一 test suite 過長。
