# 核心模組分析

> 分析日期：2026-04-10  
> 來源：[sources/openclaw/src](../../sources/openclaw/src)

---

## 模組依賴關係總覽

```
┌──────────────────────────────────────────────────────┐
│                    Gateway Server                    │
│   (WebSocket control plane + HTTP endpoints)         │
│                                                      │
│  server-methods/                                     │
│   chat │ sessions │ channels │ config │ nodes │ ...  │
└────────────────────┬─────────────────────────────────┘
                     │ 呼叫 / 訂閱
          ┌──────────┴──────────┐
          │                     │
┌─────────▼──────────┐  ┌───────▼────────────┐
│  Agent Runtime     │  │  Channel Layer      │
│  (src/agents/)     │  │  (src/channels/)    │
│                    │  │                     │
│  pi-embedded-      │  │  allowlist          │
│  runner            │  │  mention-gating     │
│  ├── run.ts        │  │  typing indicator   │
│  ├── model.ts      │  │  reply-prefix       │
│  ├── compaction    │  │  session binding    │
│  ├── skills        │  └─────────────────────┘
│  └── subagent-     │
│     registry       │
└────────────────────┘
          │
    ┌─────┴──────┐
    │  Sessions  │
    │(src/sessions/)
    │  session-id│
    │  transcript│
    │  lifecycle │
    └────────────┘
```

---

## 1. Gateway Server（`src/gateway/`）

**職責**：整個系統的 control plane，所有客戶端（CLI、WebChat、macOS app、iOS/Android node）都透過 WebSocket 連線到此。

### 核心入口

```
server.ts           → lazy-load gateway，避免啟動時間過長
server.impl.ts      → 實際 gateway 啟動邏輯（import 約 60 個子模組）
server.ts → startGatewayServer()
```

### WebSocket Method 路由

`server-methods.ts` 將所有 WS method 集合為一個 handler map，結構如下：

```typescript
export const coreGatewayHandlers: GatewayRequestHandlers = {
  ...connectHandlers,     // 連線握手、role 驗證
  ...chatHandlers,        // 訊息發送與接收
  ...sessionsHandlers,    // session 管理
  ...channelsHandlers,    // channel 狀態
  ...configHandlers,      // 設定讀寫
  ...nodeHandlers,        // 裝置節點（macOS/iOS/Android）
  ...cronHandlers,        // Cron job
  ...skillsHandlers,      // Skills 管理
  ...modelsHandlers,      // 模型目錄
  ...talkHandlers,        // Talk mode（語音）
  ...voicewakeHandlers,   // Voice wake
  // ... 共約 25 個 handler 群組
}
```

### 授權模型

```typescript
// 角色分層
role: "operator" | "node" | "admin"

// 每個 method 都要通過：
// 1. role 檢查（isRoleAuthorizedForMethod）
// 2. scope 檢查（authorizeOperatorScopesForMethod）
// 3. rate limit（consumeControlPlaneWriteBudget）
```

Gateway 的 `server-methods/` 子目錄有 25+ 個 handler 模組，每個對應一個 WS method 命名空間（`chat.*`、`sessions.*`、`config.*` 等）。

### 啟動流程（server.impl.ts 的職責拆解）

| 階段 | 模組 | 說明 |
|------|------|------|
| Config 載入 | `config/config.js` | 讀取 `~/.openclaw/openclaw.json` |
| Plugin bootstrap | `server-startup-plugins.ts` | 載入 channel/provider plugins |
| Runtime services 啟動 | `server-runtime-services.ts` | cron、webhooks、memory 等 |
| WS handler 掛載 | `server-ws-runtime.ts` + `attachGatewayWsHandlers` | 綁定所有 WS method |
| Config reload 監聽 | `server-reload-handlers.ts` | 設定熱重載 |
| Tailscale | `server-tailscale.ts` | 可選的 Tailscale Serve/Funnel |

---

## 2. Agent Runtime（`src/agents/`）

**職責**：驅動 LLM 呼叫、工具執行、multi-agent 協調的核心引擎。這是整個系統最複雜的模組。

### 2.1 Pi Embedded Runner（`src/agents/pi-embedded-runner/`）

「Pi」是 OpenClaw 對 Claude CLI（claude-cli）的內部稱呼；「embedded」指的是 agent 直接在 Gateway process 內執行，而非 fork 一個子行程。

```
pi-embedded-runner/
├── run.ts              ← 主執行迴圈（關鍵入口）
├── runs.ts             ← 執行中 run 的 registry（abort、queue、wait）
├── model.ts            ← 非同步 model 選擇與解析
├── compact.ts          ← Context compaction（摘要以釋放 token）
├── compaction-runtime-context.ts  ← Compaction 的 runtime 上下文
├── lanes.ts            ← 並行度控制（global lane / session lane）
├── skills-runtime.ts   ← Skills 的 runtime 載入
├── system-prompt.ts    ← System prompt 組合（per-run）
├── extra-params.ts     ← Provider 特有參數（OpenRouter、Ollama...）
├── history.ts          ← DM history / turn limit
├── types.ts            ← EmbeddedPiRunMeta、EmbeddedPiRunResult
└── run/
    ├── attempt.ts          ← 單次 LLM API 呼叫嘗試
    ├── assistant-failover.ts ← Failover 決策執行
    ├── auth-controller.ts    ← Auth profile 輪替
    └── failover-policy.ts    ← Failover 策略（retry/switch model）
```

**核心執行流（run.ts）**：

```
runEmbeddedPiAgent()
  ├── resolveModelAsync()         // 選擇 provider + model
  ├── enqueueCommandInLane()      // 排入 lane（控制並行）
  ├── loop {
  │     runEmbeddedAttempt()      // 呼叫 LLM API
  │     ├── 成功 → 結束 or 繼續（tool use loop）
  │     ├── context overflow → compact → retry
  │     ├── auth failure → rotate auth profile → retry  
  │     └── fatal error → failover to next model → retry
  │   }
  └── 回傳 EmbeddedPiRunResult
```

### 2.2 Subagent Registry（`src/agents/subagent-registry.ts`）

管理所有 subagent 的生命週期：

```typescript
type SubagentRunRecord = {
  runId: string
  sessionKey: string        // 對應的 session
  requesterSessionKey: string   // 發起者 session
  status: "pending" | "running" | "completed" | "error" | "killed"
  // ...
}
```

**Dependency injection 設計**：

```typescript
type SubagentRegistryDeps = {
  callGateway: typeof callGateway        // 透過 WS 與 Gateway 通訊
  persistSubagentRunsToDisk: ...         // 狀態持久化
  restoreSubagentRunsFromDisk: ...       // crash 後恢復
  runSubagentAnnounceFlow: ...           // 結果回傳給發起者
  // ...
}
```

這讓測試可以注入 mock deps，不需要跑真實 Gateway。

### 2.3 System Prompt 組合（`src/agents/system-prompt.ts`）

System prompt 的組裝順序由 `CONTEXT_FILE_ORDER` 決定：

```typescript
const CONTEXT_FILE_ORDER = new Map([
  ["agents.md",    10],   // 使用者的 agent 指引
  ["soul.md",      20],   // 助理的性格/身份
  ["identity.md",  30],   // 快速 identity 片段  
  ["user.md",      40],   // 使用者個人化設定
  ["tools.md",     50],   // 工具說明
  ["bootstrap.md", 60],   // bootstrap 指引
  ["memory.md",    70],   // memory 內容
])
```

`PromptMode` 控制注入程度：
- `"full"` — 主 agent，所有 section
- `"minimal"` — subagent，只注入 Tooling/Workspace/Runtime
- `"none"` — 只有基本 identity 一行

### 2.4 Skills Runtime（`src/agents/pi-embedded-runner/skills-runtime.ts`）

```
workspace/
└── skills/
    └── <skill-name>/
        └── SKILL.md   ← 注入 system prompt 的指引
```

Skills 在 run 時被 `resolveSkillsPromptForRun()` 載入，合併進 bootstrap prompt section。

### 2.5 Compaction（Context 壓縮）

當 context window 接近上限時，runner 會觸發 compaction：

```
原始對話記錄
    │
    ▼
summarize session（用 LLM 產生摘要）
    │
    ▼
以摘要替換舊訊息，釋放 token
    │
    ▼
繼續執行
```

重試機制：`compact-reasons.ts` 分類 compaction 失敗原因；`compaction-safety-timeout.ts` 設置 compaction 的最大等待時間。

---

## 3. Channel Layer（`src/channels/`）

**職責**：定義所有 channel 共通的行為抽象，具體 channel 實作在 `extensions/`。

### 核心抽象

```typescript
// 訊息路由到哪個 session
recordInboundSession({
  sessionKey,       // 決定路由目標
  ctx,              // 訊息上下文
  updateLastRoute,  // 更新回覆管道
})
```

### Channel 共通行為模組

| 模組 | 功能 |
|------|------|
| `allow-from.ts` | allowlist 驗證（誰可以傳訊息給 bot） |
| `mention-gating.ts` | 群組中需要 @mention 才觸發 |
| `typing.ts` + `typing-start-guard.ts` | typing indicator，防止重複觸發 |
| `ack-reactions.ts` + `status-reactions.ts` | 訊息回應（已讀 reaction / 狀態 emoji） |
| `thread-binding-policy.ts` | 訊息與 thread 的綁定規則 |
| `inbound-debounce-policy.ts` | 防抖：避免短時間內重複處理同一訊息 |
| `conversation-label.ts` | 對話標籤顯示 |
| `draft-stream-controls.ts` | 串流回覆的 draft 控制 |
| `run-state-machine.ts` | Channel 層的狀態機 |

---

## 4. Session Model（`src/sessions/`）

**職責**：定義 session 的 ID 系統、lifecycle、transcript 事件。

### Session Key 設計

Session key 是路由決策的核心：

```
格式：<channel>:<account-id>:<conversation-id>

範例：
  telegram:123456789          ← DM（user ID）
  discord:guild-id:channel-id ← 群組頻道
  main                        ← 主 session（Gateway 直連）
```

### Session Lifecycle Events

```typescript
// transcript-events.ts
type TranscriptEvent =
  | { type: "user_message"; ... }
  | { type: "assistant_message"; ... }
  | { type: "tool_call"; ... }
  | { type: "tool_result"; ... }
  | { type: "session_compacted"; ... }
```

### Send Policy（`send-policy.ts`）

控制 assistant 的回覆策略：
- `"immediate"` — 立即回覆
- `"queue"` — 排隊回覆（避免並發）
- `"drop"` — 忽略（用於靜默模式）

---

## 5. Plugin System（`packages/plugin-sdk/`）

**職責**：讓 third-party 或 extension 可以安全地擴展 agent 行為。

### Plugin Hook 類型

```typescript
// 每個 plugin 可以 hook 以下生命週期
interface OpenClawPlugin {
  // Agent 生命週期
  onBeforeChat?(ctx): Promise<void>
  onAfterChat?(ctx): Promise<void>
  
  // Tool 生命週期
  onBeforeToolCall?(ctx): Promise<ToolCallResult | void>
  onAfterToolCall?(ctx): Promise<void>
  
  // Gateway 生命週期
  onGatewayStart?(ctx): Promise<void>
  onGatewayStop?(ctx): Promise<void>
}
```

### Plugin 分類

| 類型 | 說明 | 範例 |
|------|------|------|
| Channel plugin | 接入通訊平台 | `extensions/telegram/`、`extensions/discord/` |
| Provider plugin | 接入 LLM API | `extensions/anthropic/`、`extensions/openai/` |
| Tool plugin | 提供工具給 agent | `extensions/browser/`、`extensions/diffs/` |
| Memory plugin | 提供記憶功能 | `extensions/memory-core/`、`extensions/memory-lancedb/` |

Plugin 透過 `packages/plugin-package-contract/` 定義的 schema 聲明自己的依賴與能力。

---

## 6. Auto-Reply（`src/auto-reply/`）

**職責**：協調從收到訊息到發出回覆的完整流程。

```
inbound message
    │
    ▼
channel plugin (allowlist check)
    │
    ▼
auto-reply core
    ├── 建立/查找 session
    ├── 排入 reply queue
    └── 呼叫 agent runtime (pi-embedded-runner)
            │
            ▼
        LLM API
            │
            ▼
        reply delivery（透過 channel plugin 回傳）
```

### 關鍵子模組

| 模組 | 功能 |
|------|------|
| `thinking.ts` | `think level` 控制（off/minimal/low/medium/high/xhigh） |
| `tokens.ts` | 特殊 token（如 `SILENT_REPLY_TOKEN`，表示不回覆） |
| `templating.ts` | `MsgContext` 訊息模板上下文 |

---

## 7. Process/Command Queue（`src/process/`）

**職責**：控制 agent 呼叫的並行。

```typescript
// Lane 設計：避免同一 session 並行執行
enqueueCommandInLane(laneId, task)

// Global lane：限制整體並發數
resolveGlobalLane()

// Session lane：同一 session 排隊執行
resolveSessionLane(sessionKey)
```

這確保同一個對話的訊息按順序處理，不會有 race condition。

---

## 設計模式觀察

### 1. Lazy Static Import（Gateway）
`server.ts` 透過 `loadServerImpl()` 動態 import `server.impl.ts`，避免啟動時載入所有 60+ 依賴模組，加快 CLI 的初始回應時間。

### 2. Dependency Injection via Type（Subagent Registry）
所有外部依賴以 `Deps` type 傳入，預設值指向真實模組，測試時注入 mock。這是無需 DI framework 的輕量 DI 模式。

### 3. Lane-based Concurrency（Process Queue）
不使用 mutex 或 semaphore，而是把任務排入有名稱的 lane queue，天然地控制同一 session 的訊息序列化。

### 4. Prompt as Data（System Prompt）
System prompt 的各個 section 由 `CONTEXT_FILE_ORDER` 排序，可以 per-run 組合。SKILL.md、AGENTS.md、SOUL.md 等都是 prompt data，不是 code logic。

### 5. Event-driven Channel State
Channel 的狀態（typing、reactions、presence）透過 event 驅動，而不是同步呼叫，讓 channel 行為與 agent 執行解耦。
