> 譯自 [原始 README](../../sources/codex/README.md)

# Codex CLI — 深度研究筆記

## 1. 專案簡介

**Codex CLI** 是 OpenAI 開源的本地端 coding agent，以 CLI（命令列介面）形式執行於使用者電腦上。

- **用途**：在本地終端機中完成程式碼撰寫、修改、搜尋、執行等任務，透過 LLM 驅動的 agent loop 自動操作工具。
- **特色**：
  - 跨平台沙箱（macOS Seatbelt / Linux Landlock+Bubblewrap / Windows Restricted Token）
  - 細粒度工具呼叫授權機制（approval policy）
  - 支援 MCP（Model Context Protocol）工具整合
  - 支援 multi-agent 子 agent 並行架構
  - 支援 ChatGPT Plus/Pro 帳號直接登入使用（不一定需要 API key）
- **版本**：monorepo，主要包含 `codex-cli`（TypeScript，輕量包裝層）與 `codex-rs`（Rust，核心實作）。
- **安裝**：`npm install -g @openai/codex` 或 `brew install --cask codex`

---

## 2. 整體目錄架構

```
sources/codex/
├── codex-cli/          # TypeScript 輕量 CLI 封裝（npm 發布用）
│   ├── bin/            # 入口點 binary
│   └── scripts/        # 建構腳本
├── codex-rs/           # Rust 核心實作（主要邏輯）
│   ├── core/           # 核心 agent loop、tool 系統、sandbox 整合
│   │   ├── src/agent/  # sub-agent 生命週期、mailbox、role
│   │   ├── src/tools/  # 所有 tool handler 實作
│   │   ├── src/sandboxing/ # sandbox 整合層（policy apply）
│   │   ├── src/codex.rs    # 主要 Codex session 物件
│   │   ├── src/codex_thread.rs # CodexThread：對話驅動介面
│   │   ├── gpt-5.1-codex-max_prompt.md  # GPT-5.1 系統 prompt
│   │   ├── gpt_5_2_prompt.md            # GPT-5.2 系統 prompt
│   │   └── prompt_with_apply_patch_instructions.md # apply_patch 版 prompt
│   ├── sandboxing/     # 平台 sandbox 原語（Seatbelt / Landlock / bwrap）
│   ├── exec/           # 外部執行（exec）CLI 與事件處理
│   ├── exec-server/    # exec server（長駐行程，管理 env）
│   ├── execpolicy/     # exec policy 規則引擎（rules 檔案）
│   ├── tui/            # TUI（終端 UI）Ratatui 實作
│   ├── cli/            # Rust CLI 入口
│   ├── protocol/       # 訊息協定型別（Op / Event / SandboxPolicy）
│   ├── tools/          # Tool spec、DiscoverableTool 定義
│   ├── apply-patch/    # apply_patch 格式解析
│   ├── mcp-server/     # MCP server 管理
│   ├── codex-mcp/      # MCP 連線管理器
│   ├── hooks/          # pre/post tool use hook 系統
│   ├── linux-sandbox/  # Linux 沙箱 helper binary
│   ├── network-proxy/  # 網路 proxy 管理
│   ├── login/          # ChatGPT OAuth / API key 認證
│   ├── rollout/        # 對話 rollout 記錄（本地持久化）
│   └── state/          # session state DB
├── docs/               # 文件（contributing、install、linux_sandbox）
├── sdk/                # SDK 相關
└── scripts/            # monorepo 維護腳本
```

---

## 3. 技術棧

| 層次 | 語言 / 框架 | 說明 |
|------|------------|------|
| 核心 | **Rust** | 所有 agent loop、tool、sandbox、TUI 邏輯 |
| CLI 封裝 | **TypeScript / Node.js** | npm 套件包裝，呼叫 Rust binary |
| TUI | **Ratatui**（Rust） | 終端 UI |
| 非同步 runtime | **Tokio** | async Rust 任務調度 |
| LLM 呼叫 | OpenAI Responses API / Realtime API | 支援 o 系列、GPT-4o、GPT-5 |
| MCP | **rmcp**（Rust MCP 客戶端） | 第三方工具整合 |
| Sandbox（macOS） | **Apple Seatbelt**（`/usr/bin/sandbox-exec`） | SBPL 策略語言 |
| Sandbox（Linux） | **Landlock + Bubblewrap（bwrap）** | 核心 LSM + 容器隔離 |
| Sandbox（Windows） | **Restricted Token** | 降權執行 |
| 建構系統 | **Cargo**（Rust）+ **Bazel**（跨平台 CI）+ **pnpm** | monorepo |
| 搜尋 | **BM25**（Rust） | tool_search 語意搜尋 |
| 遙測 | **OpenTelemetry（OTEL）** | span / trace |
| Hook | 自定義 `codex-hooks` crate | pre/post tool use 事件 |

---

## 4. 核心模組說明

### `codex-rs/core/src/codex.rs` — Codex Session
主要 session 物件，持有：
- model 設定、approval policy、sandbox policy
- tool registry
- MCP manager
- 對話 rollout 記錄器
- 收發 `Op`（輸入操作）與 `Event`（輸出事件）的 async channel

### `codex-rs/core/src/codex_thread.rs` — CodexThread
對話執行緒包裝，提供：
- `submit(op)` — 送出使用者輸入
- `steer_input()` — 即時引導（interrupt / continue）
- rollout 持久化 flush

### `codex-rs/core/src/agent/` — Agent 子系統
- `control.rs`：生命週期（spawn / wait / close / resume），支援 sub-agent fork
- `mailbox.rs`：agent 間訊息傳遞
- `role.rs`：角色設定（預設 `default`，可自訂 role config）
- `status.rs`：agent 狀態機（Idle → Running → Done / Error）
- `registry.rs`：活躍 agent 追蹤、深度限制（spawn depth limit）

### `codex-rs/core/src/tools/` — Tool 系統
- `spec.rs`：用 handler 類型建立 `ToolRegistryBuilder`，組裝所有可用 tool
- `registry.rs`：`ToolHandler` trait、pre/post hook 呼叫、`ApprovalStore` 快取
- `orchestrator.rs`：approval → sandbox 選擇 → 執行 → 重試 的驅動序列
- `sandboxing.rs`：`ToolRuntime` trait、`ApprovalCtx`、`SandboxAttempt`
- `handlers/`：各 tool 的具體實作（見第 7 節）

### `codex-rs/sandboxing/` — 平台 Sandbox 原語
- `seatbelt.rs`：組裝 macOS SBPL 策略字串、呼叫 `/usr/bin/sandbox-exec`
- `landlock.rs`：Linux sandbox helper CLI 參數建構
- `bwrap.rs`：Bubblewrap 容器啟動
- `manager.rs`：`SandboxManager`，統一各平台的 `SandboxCommand` 建構

### `codex-rs/exec/` — 外部執行層
- `event_processor.rs`：解析 LLM 串流事件、判斷 tool call / text
- `main.rs`：`codex exec` 子命令入口（非互動式單次執行）

### `codex-rs/execpolicy/` — 執行策略引擎
- 以 `.rules` 檔案定義允許/拒絕規則，前綴白名單、網路規則
- 整合到 `exec_policy.rs`，動態決定 shell 指令是否需要 approval

### `codex-rs/tui/` — 終端 UI
- Ratatui 實作，分 chat pane / bottom pane（輸入框、footer）
- 動態顯示 agent 任務進度、approval 詢問、sandbox 狀態

---

## 5. Agent Loop 設計

Codex 的 agent loop 以 **Tokio async task** 實作，核心流程如下：

```
使用者輸入
    │
    ▼
Op::UserMessage ──► Codex::submit()
    │
    ▼
構建 LLM request（system prompt + 對話歷史 + tool definitions）
    │
    ▼
呼叫 OpenAI Responses API（串流）
    │
    ├── text delta ──► 輸出給 TUI
    │
    └── function_call ──► ToolOrchestrator
            │
            ▼
        ① approval 檢查（ApprovalStore 快取 or 詢問使用者）
            │
            ▼
        ② sandbox 選擇（平台 sandbox 或 DangerFullAccess）
            │
            ▼
        ③ 執行 tool
            │
            ▼
        ④ 網路 approval（deferred / immediate）
            │
            ▼
        tool 結果回送 model
            │
            ▼
    model 繼續串流（回到上層 LLM request）
    │
    ▼
LLM 停止輸出（finish_reason = stop）
    │
    ▼
emit Event::AgentDone
```

**停止條件**：
- LLM 回傳 `finish_reason: stop`（無更多 tool call）
- `AgentStatus` 變為 terminal 狀態（Done / Error）
- 使用者中斷（interrupt Op）
- Guardian（approval 審查員）拒絕某工具呼叫

**對話壓縮（compact）**：
- 當 context 超出限制，自動執行 inline auto-compact 或 remote compact task，裁剪歷史後繼續

---

## 6. Sandbox 機制

Codex 的沙箱設計是其最核心的安全特色，**依平台選擇不同機制**：

### macOS — Apple Seatbelt（sandbox-exec）

- 使用 `/usr/bin/sandbox-exec -f <policy>` 包裹子行程
- 策略語言：SBPL（Sandbox Profile Language），靈感來自 Chrome sandbox policy
- 基礎策略 (`seatbelt_base_policy.sbpl`)：**預設全部拒絕**（`(deny default)`），只開放：
  - 明確列出的 sysctl 讀取
  - process fork / exec（同 sandbox 內）
  - 有限度的 file 讀取（以 writable roots 為準）
- 網路策略 (`seatbelt_network_policy.sbpl`)：可選擇允許全部網路、只允許 proxy port、或完全禁止
- `restricted_read_only_platform_defaults.sbpl`：只讀模式 platform defaults

### Linux — Landlock + Bubblewrap（bwrap）

- 使用 `codex-linux-sandbox` helper binary（自我呼叫 `arg0` 機制）
- `Landlock`：Linux kernel LSM，限制 file system 存取（按目錄白名單）
- `Bubblewrap`（bwrap）：user namespace 容器化，進一步隔離 mount namespace
- 策略透過 JSON CLI 參數傳給 helper：`--sandbox-policy`、`--file-system-sandbox-policy`、`--network-sandbox-policy`
- 支援 `--allow-network-for-proxy`：managed network 模式只允許 proxy 流量

### Windows — Restricted Token

- 降低執行 process 權限（Restricted Token）
- 可設定 `WindowsSandboxLevel`（Disabled / Low / High）

### SandboxType 枚舉

```rust
pub enum SandboxType {
    None,              // DangerFullAccess 模式
    MacosSeatbelt,
    LinuxSeccomp,      // Landlock + bwrap
    WindowsRestrictedToken,
}
```

### SandboxablePreference

- `Auto`：自動選擇平台沙箱
- `Require`：強制要求沙箱（否則拒絕執行）
- `Forbid`：禁用沙箱（DangerFullAccess）

### 環境變數標記

- `CODEX_SANDBOX=seatbelt` — 當前在 Seatbelt 沙箱中
- `CODEX_SANDBOX_NETWORK_DISABLED=1` — 網路已被沙箱停用

---

## 7. 支援的 Tools

Codex 的 tool 系統以 `ToolHandler` trait 統一介面，以下為完整清單：

| Tool 名稱 | Handler | 功能說明 |
|-----------|---------|---------|
| `shell` | `ShellHandler` | 執行 shell 指令（以使用者 shell 執行，有 sandbox） |
| `shell_command` | `ShellCommandHandler` | 執行單一 shell 指令（classic / zsh-fork backend） |
| `apply_patch` | `ApplyPatchHandler` | 套用自訂 diff patch 格式修改檔案 |
| `list_dir` | `ListDirHandler` | 列出目錄內容（支援 depth / offset / limit） |
| `view_image` | `ViewImageHandler` | 讀取圖片檔案並送入 multimodal model |
| `update_plan` | `PlanHandler` | 更新任務計畫（顯示在 TUI 中） |
| `request_user_input` | `RequestUserInputHandler` | 向使用者請求輸入（只有 root thread 可用） |
| `request_permissions` | `RequestPermissionsHandler` | 動態申請額外檔案系統或網路權限 |
| `js_repl` | `JsReplHandler` | 執行 JavaScript（REPL 模式） |
| `js_repl_reset` | `JsReplResetHandler` | 重置 JS REPL 狀態 |
| `mcp_*` | `McpHandler` | 呼叫 MCP server 的工具（動態載入） |
| `mcp_resource_*` | `McpResourceHandler` | 讀取 MCP server 的 resource |
| `tool_search` | `ToolSearchHandler` | BM25 語意搜尋已載入的 MCP tools |
| `tool_suggest` | `ToolSuggestHandler` | 建議安裝適合的 MCP tool |
| `spawn_agent` | `SpawnAgentHandler` / `SpawnAgentHandlerV2` | 建立子 agent（multi-agent 模式） |
| `wait_agent` | `WaitAgentHandler` / `WaitAgentHandlerV2` | 等待子 agent 完成 |
| `send_input` / `send_message` | `SendInputHandler` / `SendMessageHandlerV2` | 送訊息給子 agent |
| `resume_agent` | `ResumeAgentHandler` | 恢復暫停的子 agent |
| `close_agent` | `CloseAgentHandler` / `CloseAgentHandlerV2` | 關閉子 agent |
| `list_agents` | `ListAgentsHandlerV2` | 列出所有活躍子 agent |
| `followup_task` | `FollowupTaskHandlerV2` | 發送後續任務給子 agent |
| `batch_job` | `BatchJobHandler` | 批次 job 管理 |
| `code_mode_execute` | `CodeModeExecuteHandler` | code mode 執行（獨立執行模式） |
| `code_mode_wait` | `CodeModeWaitHandler` | 等待 code mode 結果 |
| `unified_exec` | `UnifiedExecHandler` | 統一 exec 介面 |
| `dynamic_*` | `DynamicToolHandler` | 動態載入的自訂工具 |

---

## 8. Guardrail / Permission 機制

### AskForApproval 策略

在設定中可選擇以下授權策略：

| 值 | 行為 |
|----|------|
| `Never` | 不詢問，自動拒絕需要 approval 的操作 |
| `OnFailure` | 執行失敗後才詢問 |
| `OnRequest` | model 主動申請時才詢問 |
| `UnlessTrusted` | 除非是可信任來源，否則詢問 |
| `Granular(config)` | 細粒度控制：`sandbox_approval`、`rules` 分開設定 |

### ApprovalStore（快取機制）

- 如果使用者選擇「本次 session 全部允許」，結果會存入 `ApprovalStore`
- 後續相同 key 的 tool 呼叫直接跳過詢問
- key 以 `serde_json` 序列化，涵蓋命令內容與目標路徑

### SafetyCheck（patch 安全評估）

`assess_patch_safety()` 函式評估 apply_patch 是否安全：
- `AutoApprove`：路徑在 writable roots 內，且沙箱可用
- `AskUser`：需要詢問使用者
- `Reject`：明確拒絕（空 patch、寫入 read-only sandbox、跨越 project 邊界）

### ExecPolicy（規則型 policy）

- 讀取 `~/.codex/rules/*.rules` 檔案
- 支援 prefix 白名單（`allow_prefix`）和網路規則
- `is_known_safe_command()` 對常見安全指令快速放行
- `command_might_be_dangerous()` 對危險指令（`rm -rf` 等）標記
- **禁止直接呼叫 shell wrapper**（`bash -c`、`python -c` 等）以防沙箱穿透

### Guardrail（Guardian）

- `guardian/` 模組：支援外部 approval reviewer 整合（`ApprovalsReviewer`）
- 可設定為由外部 guardian service 審查工具呼叫
- 若 guardian 拒絕或超時，會回傳對應拒絕訊息給 model

### 網路 Approval

- `network_approval.rs`：兩種模式：
  - `Immediate`：執行前立即確認
  - `Deferred`：執行後才確認（非阻塞）
- 可設定允許的 hostname / port 白名單

---

## 9. 特色功能

### 1. 跨平台細粒度沙箱
同一套 SandboxPolicy 介面，底層分別由 macOS Seatbelt、Linux Landlock/bwrap、Windows Restricted Token 實作，是目前 open-source coding agent 中沙箱最完整的實作之一。

### 2. Multi-agent（子 agent）架構
model 可透過 `spawn_agent` tool 建立有獨立對話歷史的子 agent，支援：
- `FullHistory`：繼承完整父對話歷史
- `LastNTurns(n)`：只繼承最後 n 輪
- 並行等待多個子 agent（`wait_agent` with multiple IDs）
- 深度限制（spawn depth limit）防止無限遞迴

### 3. AGENTS.md 規範
repo 中任何層級的 `AGENTS.md` 都會被讀入作為 developer message，是 OpenAI 推廣的 agent instruction 格式。

### 4. MCP Tool 動態載入 + 工具搜尋
- 可連接多個 MCP server，動態載入工具定義
- 提供 `tool_search`（BM25）讓 model 在大量工具中快速找到正確工具

### 5. apply_patch 自訂 diff 格式
Codex 使用自訂的 patch 格式（非 unified diff），專為 LLM 輸出優化，能精確描述新增、修改、移動、刪除操作。

### 6. 對話 Rollout 持久化
每次對話都記錄到本地 rollout DB（SQLite）中，支援：
- 續接上次對話
- 壓縮（compact）長對話
- 子 agent fork 繼承歷史

### 7. ChatGPT 帳號直接登入
不需要 API key，可透過 OAuth 直接使用 ChatGPT Plus/Pro 帳號，對一般使用者友善。

### 8. Hooks 系統
支援 pre/post tool use hooks（`codex-hooks` crate），可在工具執行前後觸發自訂行為（例如：安全審計、額外記錄）。

---

## 10. README 全文翻譯

```
npm i -g @openai/codex
```
或
```
brew install --cask codex
```

**Codex CLI** 是 OpenAI 推出的 coding agent，在你的電腦本地端執行。

如果你想在程式碼編輯器中使用 Codex（VS Code、Cursor、Windsurf），請至 IDE 中安裝。
如果你想要桌面應用體驗，執行 `codex app` 或前往 Codex App 頁面。
如果你在尋找 OpenAI 的*雲端 agent* **Codex Web**，請前往 chatgpt.com/codex。

---

### 快速入門

#### 安裝與執行 Codex CLI

使用偏好的套件管理器全域安裝：

```shell
# 使用 npm 安裝
npm install -g @openai/codex
```

```shell
# 使用 Homebrew 安裝
brew install --cask codex
```

安裝後，直接執行 `codex` 即可開始使用。

你也可以前往最新的 GitHub Release 下載適合你平台的 binary：

- macOS
  - Apple Silicon/arm64：`codex-aarch64-apple-darwin.tar.gz`
  - x86_64（較舊的 Mac 硬體）：`codex-x86_64-apple-darwin.tar.gz`
- Linux
  - x86_64：`codex-x86_64-unknown-linux-musl.tar.gz`
  - arm64：`codex-aarch64-unknown-linux-musl.tar.gz`

每個壓縮包內含一個以平台命名的執行檔（例如 `codex-x86_64-unknown-linux-musl`），解壓後建議重新命名為 `codex`。

#### 使用 ChatGPT 方案

執行 `codex` 後選擇 **以 ChatGPT 登入**。我們建議使用 ChatGPT 帳號登入，以在 Plus、Pro、Business、Edu 或 Enterprise 計畫中使用 Codex。

你也可以使用 API key 使用 Codex，但需要額外設定步驟。

### 文件

- **Codex 官方文件**：https://developers.openai.com/codex
- **貢獻指南**：./docs/contributing.md
- **安裝與建構**：./docs/install.md
- **開源基金**：./docs/open-source-fund.md

本 repository 以 Apache-2.0 授權條款發布。
```

---

## 附錄：System Prompt 摘要

Codex 依模型版本載入不同 system prompt（存放於 `codex-rs/core/`）：

- `prompt_with_apply_patch_instructions.md`：預設通用 prompt（強調 `apply_patch` 使用）
- `gpt-5.1-codex-max_prompt.md`：GPT-5.1 Codex Max 版本
- `gpt_5_2_prompt.md`：GPT-5.2 版本（增加 Autonomy 指示）
- `gpt_5_codex_prompt.md`：GPT-5 Codex 版本

共同要點：
1. 角色定位：terminal-based coding assistant，開源專案
2. 工具呼叫前應送出 preamble 訊息
3. 預設使用 `apply_patch` 修改單一檔案
4. `update_plan` tool 用於複雜多步驟任務追蹤
5. 禁止 `git reset --hard`、`git checkout --` 等破壞性指令
6. 搜尋優先使用 `rg`（ripgrep）
7. frontend 工作應避免「AI slop」，追求視覺獨特性
