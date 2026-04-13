# OpenCode 核心模組分析

> 基於 [sources/opencode/packages/opencode/src](../../sources/opencode/packages/opencode/src) 分析

## Agent 層

### `agent/`

Agent 定義與 service。`Agent.Info` schema 描述 agent 的屬性：

| 屬性          | 說明                           |
| ------------- | ------------------------------ |
| `name`        | Agent 名稱                     |
| `mode`        | `primary` / `subagent` / `all` |
| `model`       | 使用的 LLM model               |
| `permission`  | 工具呼叫 permission 規則       |
| `prompt`      | 自定義 system prompt           |
| `temperature` | LLM 取樣溫度                   |

`agent/prompt/` 存放 txt 格式的 prompt 模板：`compaction`、`explore`、`summary`、`title`。

---

## Session 層（核心）

Session 是一個對話脈絡（conversation context），包含 project 歸屬、parent/child 關係、title、token 統計、share URL。

| 模組                     | 職責                                                                                                     |
| ------------------------ | -------------------------------------------------------------------------------------------------------- |
| `session/index.ts`       | Session 主 service；CRUD、狀態管理                                                                       |
| `session/llm.ts`         | `LLM.Service`：封裝 Vercel AI SDK `streamText`；支援 abort、retry、permission check                      |
| `session/processor.ts`   | `SessionProcessor`：處理 LLM stream events；管理 tool call 生命週期；決定 compact / stop / continue      |
| `session/message-v2.ts`  | 訊息格式 v2 schema（user、assistant、tool parts、file parts、error parts）                               |
| `session/system.ts`      | `SystemPrompt`：依 provider/model 選擇對應 system prompt（anthropic、gemini、gpt、kimi、codex、default） |
| `session/compaction.ts`  | Context 超出 token 限制時壓縮舊訊息為摘要                                                                |
| `session/overflow.ts`    | 偵測 token overflow 條件                                                                                 |
| `session/retry.ts`       | LLM 呼叫失敗時的 retry 邏輯                                                                              |
| `session/revert.ts`      | Session revert（復原到前一個 snapshot 狀態）                                                             |
| `session/summary.ts`     | 產生 session 摘要                                                                                        |
| `session/todo.ts`        | Session 內的 TODO list 管理                                                                              |
| `session/instruction.ts` | 自定義 instruction 管理                                                                                  |

---

## Provider 層

### `provider/`

整合所有 LLM provider，透過 Vercel AI SDK 的統一 `streamText` 介面呼叫。

**內建支援的 provider：**

| Provider               | 備註                           |
| ---------------------- | ------------------------------ |
| Anthropic              | Claude 系列                    |
| OpenAI                 | GPT 系列                       |
| Azure OpenAI           | Azure 托管版                   |
| Google Gemini          | Gemini 系列                    |
| Google Vertex          | Vertex AI                      |
| Amazon Bedrock         | AWS 托管                       |
| Mistral                | Mistral AI                     |
| Groq                   | 高速推論                       |
| xAI                    | Grok 系列                      |
| Cohere                 | Command 系列                   |
| Perplexity             | Sonar 系列                     |
| DeepInfra              | 開源模型托管                   |
| Cerebras               | 高速推論晶片                   |
| Together AI            | 開源模型托管                   |
| OpenRouter             | 多 provider 路由               |
| GitLab                 | GitLab 整合 AI                 |
| Vercel                 | Vercel AI                      |
| Venice                 | Venice AI                      |
| GitHub Copilot         | 自行實作 OpenAI-compatible SDK |
| 任意 OpenAI-compatible | 自定義 endpoint                |

| 模組                      | 職責                                                              |
| ------------------------- | ----------------------------------------------------------------- |
| `provider/provider.ts`    | `Provider.Service`：provider 列表、model 選擇、呼叫路由           |
| `provider/models.ts`      | 從 models.dev 拉取可用 model 清單                                 |
| `provider/auth.ts`        | Provider auth key 驗證                                            |
| `provider/transform.ts`   | `ProviderTransform`：request/response 轉換、設定 OUTPUT_TOKEN_MAX |
| `provider/sdk/copilot.ts` | GitHub Copilot 專用 OpenAI-compatible provider                    |

---

## Tool 層

所有工具均有對應的 `.txt` prompt 說明檔，建立 tool schema 時注入。

### 檔案操作工具

| 工具             | 功能                                       |
| ---------------- | ------------------------------------------ |
| `bash.ts`        | 執行 shell command（含 timeout、輸出截斷） |
| `read.ts`        | 讀取檔案內容                               |
| `write.ts`       | 寫入 / 建立檔案                            |
| `edit.ts`        | 精確 string replace 編輯                   |
| `multiedit.ts`   | 批次多處編輯                               |
| `apply_patch.ts` | 套用 unified diff patch                    |
| `glob.ts`        | Glob pattern 檔案搜尋                      |
| `grep.ts`        | 文字搜尋（呼叫 ripgrep）                   |
| `ls.ts`          | 目錄列表                                   |

### 語言服務工具

| 工具            | 功能                                       |
| --------------- | ------------------------------------------ |
| `lsp.ts`        | LSP 查詢（hover、diagnostics、references） |
| `codesearch.ts` | 程式碼語意搜尋                             |

### 網路工具

| 工具           | 功能         |
| -------------- | ------------ |
| `webfetch.ts`  | 抓取網頁內容 |
| `websearch.ts` | Web 搜尋     |

### Agent 控制工具

| 工具          | 功能                                            |
| ------------- | ----------------------------------------------- |
| `task.ts`     | 啟動 subagent 執行子任務（agent orchestration） |
| `skill.ts`    | 載入並套用 SKILL.md 技能文件                    |
| `plan.ts`     | 進入 / 離開「plan mode」                        |
| `question.ts` | 向使用者提問（互動確認）                        |
| `todo.ts`     | 讀取 / 更新 TODO 清單                           |

### 工具管理

| 模組               | 職責                                                                                                  |
| ------------------ | ----------------------------------------------------------------------------------------------------- |
| `tool/registry.ts` | `ToolRegistry.Service`：動態組合可用工具清單（考量 agent 設定、provider 能力、plugin 貢獻的自訂工具） |

---

## 通訊 / 協定層

| 模組               | 協定                   | 職責                                                                             |
| ------------------ | ---------------------- | -------------------------------------------------------------------------------- |
| `server/server.ts` | HTTP / SSE / WebSocket | Hono server，REST API + 即時事件流；basic auth 保護；mDNS 服務發現               |
| `bus/`             | pub/sub                | Effect PubSub 事件匯流排；instance-scoped 與 global-scoped                       |
| `sync/`            | Event sourcing         | 事件版本化儲存於 SQLite；支援投影（projector）與事件重播                         |
| `acp/`             | JSON-RPC over stdio    | ACP（Agent Client Protocol）實作；讓 opencode 作為 ACP-compatible agent server   |
| `mcp/`             | MCP                    | MCP（Model Context Protocol）client；支援 stdio、SSE、StreamableHTTP；OAuth 鑑權 |
| `lsp/`             | LSP                    | LSP（Language Server Protocol）client；管理多個語言 server                       |

---

## 存儲層

| 模組                        | 職責                                                  |
| --------------------------- | ----------------------------------------------------- |
| `storage/db.ts`             | SQLite 抽象（bun-sqlite / better-sqlite3 依環境切換） |
| `storage/schema.ts`         | drizzle-orm schema 定義                               |
| `storage/storage.ts`        | 檔案系統 storage 抽象                                 |
| `storage/json-migration.ts` | 舊版 JSON config 遷移                                 |

---

## 版本控制 / Snapshot 層

| 模組        | 職責                                                                      |
| ----------- | ------------------------------------------------------------------------- |
| `snapshot/` | 以 git 機制（隔離 git repo）追蹤每次 tool 執行前後的檔案狀態；支援 revert |
| `worktree/` | git worktree 管理；支援同一 project 的多個平行工作分支                    |
| `git/`      | git 操作封裝                                                              |

---

## Guardrail 機制

### Permission System（`permission/`）

每次 tool call 執行前都經過 permission 評估，三段式決策：

| Action  | 行為                                                       |
| ------- | ---------------------------------------------------------- |
| `allow` | 直接放行                                                   |
| `deny`  | 拒絕並拋出 `DeniedError`，將相關 rule 回饋給 LLM           |
| `ask`   | 暫停執行，發出 `permission.asked` bus event 等待使用者回覆 |

使用者回覆選項：

- `once`：本次允許
- `always`：永久允許（寫入 approval 名單）
- `reject`：拒絕；可附加 feedback，LLM 會收到 `CorrectedError` 包含 feedback 文字

**Rule 格式**：`{ permission: string, pattern: string, action: "allow" | "deny" | "ask" }`，以 Wildcard 比對工具名稱（`permission`）與操作對象（`pattern`）。Ruleset 可在 session 建立時或 agent 定義層設置；無規則命中時預設為 `ask`。

### Token / Output 上限

- `OUTPUT_TOKEN_MAX` 預設 **32,000**，可透過環境變數 `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` 覆蓋
- 每次 LLM 呼叫強制套用 `maxOutputTokens`（取模型上限與 `OUTPUT_TOKEN_MAX` 的較小值）

### Context Overflow 偵測（`session/overflow.ts`）

當已用 token 接近模型 context window 上限時，自動觸發 `session/compaction.ts` 壓縮舊訊息為摘要：

- `COMPACTION_BUFFER = 20,000` tokens 預留緩衝（可透過 `config.compaction.reserved` 覆蓋）
- 可透過 `config.compaction.auto = false` 停用自動 compaction

---

## 快取機制

### Prompt Cache / KV Cache（`provider/transform.ts` — `applyCaching()`）

`ProviderTransform` 在組裝 LLM 請求時，自動對 system messages 與最近 2 輪對話注入 provider 特定的 cache control metadata：

| Provider          | Cache 注入方式                                 |
| ----------------- | ---------------------------------------------- |
| Anthropic         | `cacheControl: { type: "ephemeral" }`          |
| Amazon Bedrock    | `cachePoint: { type: "default" }`              |
| OpenAI            | `promptCacheKey = sessionID`                   |
| OpenRouter        | `cacheControl: { type: "ephemeral" }`          |
| OpenAI-compatible | `cache_control: { type: "ephemeral" }`         |
| GitHub Copilot    | `copilot_cache_control: { type: "ephemeral" }` |

Cache key 以 `sessionID` 為基礎，確保同一對話內的連續請求最大化命中 provider 端 KV cache，降低延遲與費用。

### Token 用量追蹤（`session/index.ts`）

精確追蹤三類 token 費用：

- `inputTokens`：實際計費 input（已扣除 cache 命中部分）
- `cache.read`：命中快取的 token 數（費率較低）
- `cache.write`：寫入快取的 token 數

支援來源：Anthropic、Bedrock、Google Vertex、Venice、GitHub Copilot（`input_tokens_details.cached_tokens`）。

### File Scan Cache（`file/index.ts`）

- 目錄掃描結果以 `Effect.cached` 快取，避免重複 I/O
- 偵測到檔案異動（watch）後自動失效並重新掃描

---

## 其他模組

| 模組             | 職責                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------- |
| `permission/`    | Tool 呼叫的 permission 評估（基於 ruleset）                                                       |
| `skill/`         | SKILL.md 文件探索與載入（支援 `.claude`、`.agents` 等外部格式）                                   |
| `config/`        | 設定檔讀取（JSONC 格式；全域與 local scope）                                                      |
| `auth/`          | Provider API key 的存取管理（OAuth、API key、WellKnown）                                          |
| `project/`       | Project 與 instance 抽象（project = 目錄；instance = 執行個體）                                   |
| `pty/`           | PTY（pseudo terminal）管理；供 bash tool 使用                                                     |
| `npm/`           | npm package 動態安裝（plugin 載入機制）                                                           |
| `effect/`        | Effect library helpers（`InstanceState`、`makeRuntime`、`CrossSpawnSpawner`）                     |
| `effect/`        | 所有 service 以 Effect 的 `Layer` + `ServiceMap` 模式組織                                         |
| `global/`        | 全域路徑常數（config、data、log 目錄）                                                            |
| `flag/`          | Feature flag 管理（讀取環境變數）                                                                 |
| `env/`           | 環境變數讀取                                                                                      |
| `id/`            | ULID-based ID 生成（ascending identifiers）                                                       |
| `share/`         | Session 分享功能（產生公開 URL）                                                                  |
| `control-plane/` | Workspace 管理（雲端控制平面適配器；remote workspace、Cloudflare 等）                             |
| `installation/`  | CLI 版本與安裝路徑管理                                                                            |
| `cli/`           | yargs CLI 命令（run、serve、acp、mcp、agent、provider、models、session、import/export、debug 等） |
