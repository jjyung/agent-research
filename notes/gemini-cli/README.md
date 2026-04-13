> 譯自 [原始 README](../../sources/gemini-cli/README.md)

# Gemini CLI 深度分析筆記

---

## 1. 專案簡介

**Gemini CLI** 是 Google 開源的 terminal-first AI agent，將 Gemini 模型的能力直接帶入開發者的命令列環境。

- **版本**：`0.39.0-nightly.20260408`（nightly build）
- **授權**：Apache License 2.0
- **NPM 套件**：`@google/gemini-cli`
- **Node.js 要求**：`>=20.0.0`
- **核心特色**：
  - 免費額度：個人 Google 帳號每分鐘 60 requests、每天 1,000 requests
  - 支援 Gemini 3 模型，1M token context window
  - 內建 Google Search grounding（即時搜尋）
  - MCP（Model Context Protocol）擴充支援
  - Sandbox 隔離執行（Docker / Podman）
  - 支援互動式（interactive）與非互動式（headless/scripting）兩種模式

---

## 2. 整體目錄架構

```
gemini-cli/
├── packages/
│   ├── cli/                    # 使用者介面層：React/Ink terminal UI、input handling、slash commands
│   │   └── src/
│   │       ├── gemini.tsx      # 互動式 CLI 主入口
│   │       ├── nonInteractiveCli.ts   # 非互動式（headless）模式
│   │       ├── ui/             # React UI 元件
│   │       ├── commands/       # Slash commands (/help, /chat 等)
│   │       └── acp/            # Agent Communication Protocol 整合
│   ├── core/                   # 後端核心：API 呼叫、tool 執行、agent loop
│   │   └── src/
│   │       ├── agent/          # AgentSession、AgentProtocol、事件型別定義
│   │       ├── agents/         # LocalAgentExecutor、AgentRegistry、各 built-in agents
│   │       ├── scheduler/      # Scheduler（tool call 協調）、policy、confirmation
│   │       ├── tools/          # 所有 tool 實作
│   │       ├── context/        # ChatCompressionService、context 管理
│   │       ├── prompts/        # PromptProvider（system prompt 組裝）
│   │       ├── safety/         # Safety checker protocol（外部/內建）
│   │       ├── hooks/          # Hook 系統（BeforeTool/AfterTool/BeforeAgent 等）
│   │       ├── policy/         # ApprovalMode、PolicyDecision
│   │       ├── mcp/            # MCP OAuth 和 auth provider
│   │       ├── routing/        # Model router service
│   │       ├── config/         # Config、Storage、AgentLoopContext
│   │       ├── telemetry/      # 遙測、trace、logger
│   │       └── voice/          # 語音輸入模組
│   ├── a2a-server/             # 實驗性 Agent-to-Agent server（A2A protocol）
│   ├── sdk/                    # 程式化 SDK（嵌入 Gemini CLI 能力至其他工具）
│   ├── devtools/               # 開發者工具（網路/console inspector）
│   ├── test-utils/             # 共用測試工具
│   └── vscode-ide-companion/   # VS Code extension（IDE 伴侶）
├── integration-tests/          # E2E 整合測試
├── evals/                      # 評估測試（eval suite）
├── memory-tests/               # 記憶回歸測試
├── perf-tests/                 # 效能回歸測試
├── docs/                       # 文件
├── scripts/                    # Build / release 腳本
└── GEMINI.md                   # 給 agent 閱讀的專案 context 文件
```

---

## 3. 技術棧

| 分類 | 技術 |
|------|------|
| **語言** | TypeScript（ESM modules） |
| **Runtime** | Node.js >=20.0.0 |
| **UI 框架** | React + [Ink](https://github.com/vadimdemedes/ink)（terminal 渲染） |
| **AI SDK** | `@google/genai`（官方 Gemini SDK） |
| **套件管理** | npm workspaces（monorepo） |
| **Bundler** | esbuild |
| **測試框架** | Vitest |
| **Linting** | ESLint + Prettier |
| **Schema 驗證** | Zod、AJV |
| **A2A 協定** | `@a2a-js/sdk` |
| **MCP 協定** | Model Context Protocol（自訂實作） |

---

## 4. 核心模組說明

### 4.1 `packages/cli/src/` — 使用者介面層

| 檔案/目錄 | 職責 |
|-----------|------|
| `gemini.tsx` | 互動式 CLI 主入口，React/Ink 應用程式根元件 |
| `nonInteractiveCli.ts` | headless/scripting 模式（`-p` flag），直接輸出結果 |
| `nonInteractiveCliAgentSession.ts` | headless 模式下的 AgentSession 管理 |
| `ui/` | 所有 terminal UI 元件（輸入框、訊息列表、工具確認對話框等） |
| `commands/` | Slash commands（`/help`、`/chat`、`/clear` 等） |
| `acp/` | Agent Communication Protocol（ACP）整合，用於 agent 間通訊 |
| `config/` | CLI 層級的設定（theme、keybindings 等） |

### 4.2 `packages/core/src/agent/` — Agent 協定層

定義 agent 通訊的抽象介面。

| 檔案 | 職責 |
|------|------|
| `types.ts` | `AgentProtocol`、`AgentSend`、`AgentEvent`（所有事件型別的 discriminated union） |
| `agent-session.ts` | `AgentSession`：包裝 `AgentProtocol`，提供 `AsyncIterable<AgentEvent>` 的便利 API |

**事件型別**：`initialize`、`session_update`、`message`、`agent_start`、`agent_end`、`tool_request`、`tool_update`、`tool_response`、`elicitation_request`、`elicitation_response`、`usage`、`error`、`custom`

### 4.3 `packages/core/src/agents/` — Subagent 系統

| 檔案/目錄 | 職責 |
|-----------|------|
| `local-executor.ts` | `LocalAgentExecutor`：執行 subagent loop 的核心邏輯 |
| `registry.ts` | `AgentRegistry`：動態載入、驗證、管理所有 agent 定義 |
| `types.ts` | `LocalAgentDefinition`、`AgentTerminateMode`、`SubagentProgress` 等型別 |
| `generalist-agent.ts` | 通用型 Agent，存取所有 tools，適合重量級任務 |
| `codebase-investigator.ts` | 專門分析程式碼庫的 subagent，僅有 read-only tools，使用 HIGH thinking |
| `cli-help-agent.ts` | 回答 CLI 使用問題的 subagent |
| `memory-manager-agent.ts` | 管理記憶（GEMINI.md）的 subagent |
| `agent-tool.ts` | `AgentTool`：讓主 agent 可以呼叫 subagent 的 tool 實作 |
| `agent-scheduler.ts` | `scheduleAgentTools()`：封裝 subagent tool call 排程 |
| `a2a-client-manager.ts` | 管理遠端 A2A agent 連線 |

### 4.4 `packages/core/src/scheduler/` — Tool 執行協調器

`Scheduler` 是 tool call 的事件驅動協調引擎，負責：

1. **接收** `ToolCallRequestInfo[]`（來自模型的 function call 清單）
2. **Policy 檢查**：`checkPolicy()` 查詢 `ApprovalMode` 和 safety rules
3. **確認處理**：`resolveConfirmation()` 向使用者請求確認（若需要）
4. **Hook 執行**：`evaluateBeforeToolHook()` 執行 `BeforeTool` hooks
5. **工具執行**：`ToolExecutor` 實際執行 tool
6. **狀態管理**：`SchedulerStateManager` 追蹤每個 tool call 的狀態
7. **工具修改**：`ToolModificationHandler` 處理可修改的 tool（如 edit diff）

### 4.5 `packages/core/src/tools/` — Tool 實作層

所有 tool 繼承自 `BaseDeclarativeTool`，實例化為 `BaseToolInvocation`，核心介面：
- `shouldConfirmExecute()` — 決定是否需要使用者確認
- `execute()` — 實際執行
- `getDescription()` — 執行前的描述（顯示給使用者）
- `toolLocations()` — 影響的檔案系統路徑

### 4.6 `packages/core/src/context/` — Context 管理層

| 檔案 | 職責 |
|------|------|
| `chatCompressionService.ts` | 在 token 用量超過閾值（預設 50%）時壓縮對話歷史，保留最新 30% |
| `contextCompressionService.ts` | 更細緻的 context 壓縮策略 |
| `memoryContextManager.ts` | 管理 hierarchical memory（global + project 層級） |
| `toolDistillationService.ts` | 精煉 tool output 以節省 token |
| `toolOutputMaskingService.ts` | 遮蔽 tool output 中的敏感資訊 |
| `agentHistoryProvider.ts` | 提供 agent 的對話歷史 |

### 4.7 `packages/core/src/prompts/` — System Prompt 組裝層

`PromptProvider.getCoreSystemPrompt()` 動態組裝 system prompt，包含：
- **Preamble**：基本身份說明
- **Core Mandates**：核心操作準則
- **Primary Workflows**：主要工作流程（依 ApprovalMode 調整）
- **Planning Workflow**（Plan Mode 時）
- **Agent Skills**：已載入的 skills 清單
- **Sub-agents**：已知 subagent 的說明
- **Task Tracker**：任務追蹤目錄路徑
- **Hook Context**：hook 相關 context

支援 `GEMINI_SYSTEM_MD` 環境變數覆寫整個 system prompt。

### 4.8 `packages/core/src/hooks/` — Hook 系統

提供外部擴充點，可在特定事件前後注入自訂邏輯：

| Hook 事件 | 時機 |
|-----------|------|
| `BeforeTool` | tool 執行前 |
| `AfterTool` | tool 執行後 |
| `BeforeAgent` | subagent 啟動前 |
| `AfterAgent` | subagent 結束後 |
| `SessionStart` | session 開始 |
| `SessionEnd` | session 結束 |
| `PreCompress` | context 壓縮前 |
| `BeforeModel` | 呼叫 LLM 前 |
| `AfterModel` | LLM 回應後 |
| `BeforeToolSelection` | tool 選擇前 |

Hook 有兩種實作方式：`command`（外部命令）、`runtime`（內建函式）。Hook source 有四個層級：`project`、`user`、`system`、`extension`。

---

## 5. Agent Loop 設計

### 5.1 主 Agent Loop（互動模式）

主 agent loop 位於 `packages/cli/src/gemini.tsx` 與 `packages/core/src/` 的協作中，透過 `AgentSession.sendStream()` 驅動：

```
使用者輸入
    ↓
AgentSession.send(payload)
    ↓
GeminiChat.sendMessage()          ← 呼叫 Gemini API（streaming）
    ↓
model 回應 → functionCalls[]
    ↓
Scheduler.schedule(requests)
    ├─ checkPolicy()               ← 確認 ApprovalMode
    ├─ resolveConfirmation()       ← 如需確認，透過 MessageBus 向 CLI 請求
    ├─ evaluateBeforeToolHook()    ← 執行 BeforeTool hooks
    ├─ ToolExecutor.execute()      ← 實際執行 tool
    └─ AfterTool hooks
    ↓
tool results → 注入 conversation history
    ↓
回到 GeminiChat.sendMessage()（下一個 turn）
    ...（循環直到停止條件）
```

### 5.2 Subagent Loop（LocalAgentExecutor）

```
AgentTool.execute() 被主 agent 呼叫
    ↓
LocalAgentExecutor.create()
    ├─ 建立隔離的 ToolRegistry（clone tools，注入 subagentMessageBus）
    ├─ 強制排除 AgentTool（防止遞迴）
    └─ 強制注入 CompleteTaskTool（唯一正常停止方法）
    ↓
LocalAgentExecutor.run()
    ↓ 迴圈
    executeTurn(chat, message, turnCounter)
        ├─ tryCompressChat()          ← 超過 token 閾值時壓縮历史
        ├─ definition.onBeforeTurn()  ← agent 定義的前置 hook
        ├─ callModel()                ← 呼叫 Gemini API
        │   └─ 若 functionCalls = []  → ERROR_NO_COMPLETE_TASK_CALL
        └─ processFunctionCalls()
            └─ scheduleAgentTools()   ← 透過 Scheduler 執行
    ↓
停止條件：
    - complete_task 被呼叫 → GOAL
    - 超過 maxTurns → MAX_TURNS（觸發 Grace Period Recovery）
    - 超過 maxTimeMinutes → TIMEOUT（觸發 Grace Period Recovery）
    - abort signal → ABORTED
    - 未呼叫 complete_task 就停止 → ERROR_NO_COMPLETE_TASK_CALL（觸發 Grace Period Recovery）
```

### 5.3 Grace Period Recovery

當 subagent 因 TIMEOUT / MAX_TURNS / ERROR_NO_COMPLETE_TASK_CALL 停止時，系統給予 **1 分鐘的寬限期**，注入一條 warning message，強迫 agent 呼叫 `complete_task` 並回傳最佳結果。若失敗，直接終止。

### 5.4 Context 壓縮策略

`ChatCompressionService` 在每個 turn 前執行：
- 壓縮觸發閾值：`DEFAULT_COMPRESSION_TOKEN_THRESHOLD = 0.5`（即 token 用量超過 50% 上限）
- 保留比例：`COMPRESSION_PRESERVE_THRESHOLD = 0.3`（保留最新 30% 的對話歷史）
- Function response token 上限：`COMPRESSION_FUNCTION_RESPONSE_TOKEN_BUDGET = 50,000`

### 5.5 停止條件總結

| 情況 | 結果 |
|------|------|
| 呼叫 `complete_task` | `GOAL`（正常完成） |
| abort signal | `ABORTED` |
| 超過 `maxTurns`（預設 30） | `MAX_TURNS` → Grace Period → 若失敗強制終止 |
| 超過 `maxTimeMinutes`（預設 10 min） | `TIMEOUT` → Grace Period → 若失敗強制終止 |
| 未呼叫 tools 就停止 | `ERROR_NO_COMPLETE_TASK_CALL` → Grace Period |
| user 拒絕 tool 操作 | `ERROR`/`ABORTED` |

---

## 6. 支援的 Tools

### 6.1 檔案系統工具

| Tool 名稱 | 功能 |
|-----------|------|
| `read_file` | 讀取指定檔案（支援行號範圍） |
| `read_many_files` | 批次讀取多個檔案（支援 glob、exclude 過濾） |
| `write_file` | 寫入/建立檔案 |
| `edit` | 精確字串替換編輯（`old_string` → `new_string`，可多次替換） |
| `ls` | 列出目錄內容 |
| `glob` | 以 glob pattern 搜尋檔案 |
| `grep` | 文字/正規表示式搜尋（底層使用 ripgrep） |

### 6.2 Shell 工具

| Tool 名稱 | 功能 |
|-----------|------|
| `shell` | 執行 shell 命令（支援 `is_background` 參數） |
| `shellBackground`（內部） | 管理背景執行的 shell 程序 |

### 6.3 網路工具

| Tool 名稱 | 功能 |
|-----------|------|
| `web_search` | Google Search grounding（即時搜尋，回傳 grounding metadata） |
| `web_fetch` | 抓取任意 URL 的內容（HTTP fetch） |

### 6.4 記憶與 Context 工具

| Tool 名稱 | 功能 |
|-----------|------|
| `memory` | 儲存事實至 GEMINI.md（global 或 project scope） |
| `write_todos` | 管理任務清單（TODO list） |
| `jit_context`（JIT） | Just-In-Time 注入額外 context |
| `get_internal_docs` | 取得內部文件（AGENTS.md 等） |

### 6.5 Flow 控制工具

| Tool 名稱 | 功能 |
|-----------|------|
| `enter_plan_mode` | 進入 Plan Mode（僅允許 read-only + 計劃撰寫） |
| `exit_plan_mode` | 離開 Plan Mode，載入核准的計劃檔案 |
| `complete_task` | 宣告 subagent 任務完成（強制停止符號） |
| `ask_user` | 向使用者提問（elicitation），支援多選題 |
| `activate_skill` | 啟用特定 skill |
| `update_topic` | 更新當前 session 的主題標題 |

### 6.6 Agent 工具

| Tool 名稱 | 功能 |
|-----------|------|
| `agent` | 呼叫 subagent（如 generalist、codebase_investigator 等） |

### 6.7 MCP 工具

透過 MCP client manager 動態發現的外部工具，以 `mcp__<server_name>__<tool_name>` 命名規則掛載；支援 wildcard（`mcp__*`、`mcp__<server>__*`）。

### 6.8 Tracker 工具

| Tool 名稱 | 功能 |
|-----------|------|
| `trackerTools`（複數） | 任務追蹤（Tracker）相關操作 |

---

## 7. Guardrail / Permission 機制

### 7.1 ApprovalMode（審核模式）

定義於 `packages/core/src/policy/types.ts`，由寬鬆到嚴格排列：

| 模式 | 說明 |
|------|------|
| `PLAN` | 最嚴格：只允許 read-only 操作 + 計劃撰寫，不執行任何修改 |
| `DEFAULT` | 預設：修改型操作需要使用者確認 |
| `AUTO_EDIT` | 檔案編輯自動允許，其他破壞性操作仍需確認 |
| `YOLO` | 最寬鬆：全部自動允許，無需確認 |

### 7.2 Policy 決策流程

```
Scheduler 接到 tool call
    ↓
checkPolicy()
    ├─ 查詢 PolicyRule（可由設定檔或 safety checker 定義）
    └─ 回傳 PolicyDecision: ALLOW | DENY | ASK_USER
    ↓
若 ASK_USER → resolveConfirmation()
    ↓
ToolInvocation.shouldConfirmExecute()
    ├─ 比對 ApprovalMode
    ├─ 比對 AllowedPath（workspace 邊界）
    └─ 回傳 ToolCallConfirmationDetails | false
```

### 7.3 Safety Checker Protocol

可外掛外部 safety checker，透過 stdin/stdout 通訊：

```
SafetyCheckInput {
    protocolVersion: '1.0.0'
    toolCall: FunctionCall           ← 待確認的 tool call
    context: {
        environment: { cwd, workspaces }
        history: { turns }           ← 近期對話歷史
    }
    config?: unknown
}
```

支援兩種 checker 型別：
- `in-process`（內建）：`allowed-path`（路徑白名單）、`conseca`（自訂規則）
- `external`（外部程序）：可自行實作

### 7.4 Path Validation

`Config.validatePathAccess()` 確保 tool 存取的路徑必須在：
1. workspace 目錄範圍內，或
2. 允許的 project temp 目錄內

### 7.5 Hook 攔截

`BeforeTool` hook 可以攔截任何 tool call，回傳 ALLOW / DENY / 修改參數。

### 7.6 Trusted Folders

支援 Trusted Folders 設定，可針對特定資料夾配置不同的 execution policy。

---

## 8. 特色功能

### 8.1 Google Search Grounding

`web_search` tool 直接使用 Gemini API 的原生 grounding 功能，而非透過外部 API：
- 回傳 `GroundingMetadata`（包含 web sources、信心分數、文字片段位置）
- 讓模型的回答能引用即時資訊，並附上出處

### 8.2 多層次 Memory 系統（GEMINI.md）

Memory 具有三個 scope：
- **Global**（`~/.gemini/GEMINI.md`）：跨所有專案的個人記憶
- **Project**（`<project>/.gemini/GEMINI.md`）：專案特定 context
- **Extension**：擴充套件注入的 context

`memory` tool 可寫入、`get_internal_docs` 可讀取，系統自動在 system prompt 中附帶 context files 內容。

### 8.3 Plan Mode（計劃模式）

透過 `enter_plan_mode` / `exit_plan_mode` tools 實作：
1. 進入 Plan Mode → system prompt 切換為計劃模式（僅允許 read-only + plan 撰寫）
2. Agent 分析需求並產出計劃檔案（`.md`）
3. 使用者審核並核准計劃
4. `exit_plan_mode(planFilename)` → 載入核准計劃，切回執行模式
5. Agent 依計劃檔案逐步執行

### 8.4 Subagent 系統（Agent-in-Agent）

Built-in subagents：
- **Generalist Agent**：通用型，20 turns 上限，適合大量資料處理
- **Codebase Investigator Agent**：唯讀型，50 turns，HIGH thinking，產出結構化 JSON 報告
- **CLI Help Agent**：回答 CLI 使用問題
- **Memory Manager Agent**：管理記憶操作
- **Browser Agent**：瀏覽器自動化（實驗性）

主 agent 無法呼叫 agent（防止遞迴），但 subagent 可透過 `agent` tool 呼叫其他已知 subagents。

### 8.5 Session Checkpointing

支援對話 checkpoint，可儲存並恢復複雜的 session 狀態。

### 8.6 Context Window 壓縮策略

面對 1M token 的 context window，採用動態壓縮：
- 超過 50% token 上限時觸發壓縮
- 使用獨立的 Gemini call 生成摘要
- 保留最新 30% 對話 + 壓縮摘要注入
- 過長的 tool output 截斷後存入臨時檔案，提供 file path 供後續讀取

### 8.7 A2A（Agent-to-Agent）Protocol

實驗性功能，`packages/a2a-server/` 提供標準 A2A server，讓 Gemini CLI 可作為可被其他 agent 呼叫的遠端 agent。

### 8.8 Release Channel 管理

每週固定時程發布三個 channel：

| Channel | 發布時程 | 說明 |
|---------|----------|------|
| `stable`（latest） | 每週二 UTC 20:00 | 上週 preview + bug fixes |
| `preview` | 每週二 UTC 23:59 | 尚未完整驗證 |
| `nightly` | 每日 UTC 00:00 | main branch 每日快照 |

### 8.9 VS Code IDE Companion

`packages/vscode-ide-companion/` 提供 VS Code extension，讓 CLI 與 IDE 整合：
- IDE 狀態感知（知道哪個檔案正在編輯）
- IDE diff 視覺化

### 8.10 GitHub Action 整合

官方 GitHub Action `google-github-actions/run-gemini-cli`，支援：
- PR 自動 code review
- Issue 分類和標籤
- `@gemini-cli` mention 觸發

---

## 9. README 全文翻譯

> 以下為 gemini-cli 官方 README 的繁體中文翻譯

---

# Gemini CLI

[![Gemini CLI CI](https://github.com/google-gemini/gemini-cli/actions/workflows/ci.yml/badge.svg)](...)
[![Version](https://img.shields.io/npm/v/@google/gemini-cli)](...)
[![License](https://img.shields.io/github/license/google-gemini/gemini-cli)](...)

Gemini CLI 是一款開源 AI agent，將 Gemini 的能力直接帶入您的 terminal。它提供對 Gemini 的輕量級存取，讓您的 prompt 到模型之間的路徑最短。

詳細文件請見 [geminicli.com/docs](https://geminicli.com/docs/)。

## 為何選擇 Gemini CLI？

- **免費額度**：使用個人 Google 帳號，每分鐘 60 requests、每天 1,000 requests
- **強大的 Gemini 3 模型**：改進的推理能力與 1M token context window
- **內建工具**：Google Search grounding、檔案操作、shell 命令、web fetch
- **可擴充**：MCP（Model Context Protocol）支援自訂整合
- **Terminal-first**：為生活在命令列的開發者設計
- **開源**：Apache 2.0 授權

## 安裝

### 使用 npx 即時執行（無需安裝）

```bash
npx @google/gemini-cli
```

### 全域安裝

```bash
npm install -g @google/gemini-cli
# 或
brew install gemini-cli   # macOS/Linux
sudo port install gemini-cli  # MacPorts
```

## Release Channels

- **Preview**：每週二 UTC 23:59 發布，尚未完整驗證。`npm install -g @google/gemini-cli@preview`
- **Stable**：每週二 UTC 20:00 發布，為上週 preview 的正式版。`npm install -g @google/gemini-cli@latest`
- **Nightly**：每日 UTC 00:00 發布，為 main branch 快照。`npm install -g @google/gemini-cli@nightly`

## 主要功能

### 程式碼理解與生成

- 查詢和編輯大型 codebases
- 從 PDF、圖片或草圖生成新應用（多模態能力）
- 以自然語言除錯和排除問題

### 自動化與整合

- 自動化操作任務，如查詢 pull requests 或處理複雜的 rebase
- 使用 MCP servers 連接新能力，包含 Imagen、Veo 或 Lyria 的媒體生成
- 在腳本中以非互動式執行，實現工作流自動化

### 進階能力

- 以內建 Google Search 讓查詢具備即時資訊的 grounding
- 對話 checkpointing，儲存和恢復複雜 session
- 自訂 context 檔案（GEMINI.md）為您的專案量身調整行為

### GitHub 整合

透過 Gemini CLI GitHub Action 將 Gemini CLI 整合進您的 GitHub 工作流：
- **Pull Request Reviews**：自動化 code review
- **Issue Triage**：基於內容分析自動標籤和優先排序 issues
- **On-demand 協助**：在 issues 和 PRs 中 mention `@gemini-cli`
- **自訂工作流**：建立自動化的排程和按需工作流

## 認證選項

### 選項一：以 Google 帳號登入（OAuth）

最適合個人開發者及擁有 Gemini Code Assist License 的用戶。

- 免費額度：每分鐘 60 requests、每天 1,000 requests
- Gemini 3 模型，1M token context window
- 無需管理 API key

```bash
gemini
# 依照瀏覽器認證流程操作
```

### 選項二：Gemini API Key

最適合需要特定模型控制或付費額度存取的開發者。

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
gemini
```

### 選項三：Vertex AI

最適合企業團隊和生產工作負載，提供進階安全與合規功能。

```bash
export GOOGLE_API_KEY="YOUR_API_KEY"
export GOOGLE_GENAI_USE_VERTEXAI=true
gemini
```

## 快速開始

```bash
# 在當前目錄啟動
gemini

# 包含多個目錄
gemini --include-directories ../lib,../docs

# 使用特定模型
gemini -m gemini-2.5-flash

# 非互動式模式（腳本用）
gemini -p "解釋這個 codebase 的架構"
gemini -p "執行測試並部署" --output-format stream-json
```

## 貢獻

我們歡迎貢獻！Gemini CLI 完全開源（Apache 2.0），歡迎：回報 bugs、改善文件、提交程式碼改進、分享您的 MCP servers 和 extensions。

詳見 [CONTRIBUTING.md](./CONTRIBUTING.md)。

## 授權

Apache License 2.0
