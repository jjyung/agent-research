> 基於 [sources/codex](../../sources/codex) 分析

# Codex CLI 核心模組分析

## Agent Loop 設計

```
Op::UserMessage
  → 組裝 LLM request（OpenAI Responses API）
  → 串流解析 function_call
  → ToolOrchestrator
      ├── ApprovalStore（快取 or 詢問使用者）
      ├── Sandbox 選擇（policy-based）
      ├── 執行 tool
      └── Network approval（immediate / deferred）
  → 結果回送 model → 下一輪
  → finish_reason: stop 或使用者中斷
```

### Multi-agent 架構
- `spawn_agent`：建立 subagent（fork history 或乾淨 slate）
- `send_message` / `wait_agent`：parent 與 subagent 間通訊
- `list_agents` / `close_agent`：生命週期管理
- depth limit 防止無限遞迴

---

## 核心模組（`codex-rs/`）

### core/（主要 agent 邏輯）

| 模組 | 職責 |
|------|------|
| `src/codex.rs` | 主要 `Codex` session 物件 |
| `src/codex_thread.rs` | `CodexThread`：對話驅動介面 |
| `src/agent/` | Sub-agent 生命週期、mailbox、role 管理 |
| `src/tools/` | 所有 tool handler 實作 |
| `src/sandboxing/` | Sandbox policy 套用層 |

### sandboxing/（平台 sandbox 原語）

| 平台 | 機制 | 說明 |
|------|------|------|
| macOS | Apple Seatbelt（SBPL）| 預設全拒絕，精細 sysctl 白名單 |
| Linux | Landlock + Bubblewrap | 目錄白名單 + user namespace 容器化 |
| Windows | Restricted Token | 降權執行 |

環境變數：
- `CODEX_SANDBOX_ENV_VAR`：標記當前執行在 sandbox 內
- `CODEX_SANDBOX_NETWORK_DISABLED`：標記網路已停用

### execpolicy/（`.rules` 規則引擎）
- 讀取 `.rules` 規則檔案，決定 exec policy
- 可 ban shell wrapper，防止繞過 binary check 的沙箱穿透

### protocol/（跨模組型別）
定義 `Op`（操作指令）、`Event`（事件）、`SandboxPolicy`（沙箱策略）等跨模組共用型別。

---

## Tools（26+）

**檔案操作**

| 工具 | 功能 |
|------|------|
| `shell` | 執行 shell command（在 sandbox 內）|
| `apply_patch` | 套用自訂 diff 格式（LLM 輸出優化）|
| `list_dir` | 目錄列表 |
| `view_image` | 查看圖片 |

**Agent 控制**

| 工具 | 功能 |
|------|------|
| `update_plan` | 更新執行計劃 |
| `request_user_input` | 向使用者提問 |
| `request_permissions` | 請求額外 permission |

**Multi-agent**

| 工具 | 功能 |
|------|------|
| `spawn_agent` | 建立子 agent |
| `wait_agent` | 等待子 agent 完成 |
| `send_message` | 向子 agent 發送訊息 |
| `close_agent` | 關閉子 agent |
| `list_agents` | 列出所有活躍 agent |
| `batch_job` | 批次工作 |

**MCP 動態工具**

| 工具 | 功能 |
|------|------|
| `tool_search`（BM25）| 在大量 MCP tools 中搜尋 |
| `tool_suggest` | 推薦適合當前任務的工具 |
| `mcp_*` | 動態載入的 MCP tools |

**執行環境**

| 工具 | 功能 |
|------|------|
| `js_repl` | JavaScript REPL |

---

## Guardrail 機制

### AskForApproval（5 個層級）

| 層級 | 行為 |
|------|------|
| `Never` | 永不詢問，全自動 |
| `OnFailure` | 只在失敗時詢問 |
| `OnRequest` | 模型主動請求時詢問 |
| `UnlessTrusted` | 非信任工具才詢問 |
| `Granular` | 細粒度，逐工具逐操作詢問 |

### ApprovalStore
Session 內快取已授權的 key，避免對同一操作重複詢問。

### SafetyCheck（三判決）

| 判決 | 行為 |
|------|------|
| `AutoApprove` | 自動核准 |
| `AskUser` | 詢問使用者 |
| `Reject` | 直接拒絕 |

### Guardian
可接外部 approval reviewer service，提供自訂審查邏輯。

### ExecPolicy（`.rules` 規則引擎）
- 定義哪些 exec 指令被允許 / 禁止
- 可 ban shell wrapper 防沙箱穿透

---

## 持久化（Rollout）

使用 **SQLite** 儲存完整對話歷史（rollout），支援：
- **續接（resume）**：從上次中斷繼續
- **Compact**：壓縮歷史
- **Fork**：基於歷史建立新分支

---

## 特色功能

| 特色 | 說明 |
|------|------|
| 跨平台細粒度沙箱 | open source agent 中最完整的沙箱實作之一 |
| Multi-agent 並行 | spawn / fork history / depth limit |
| AGENTS.md 規範 | OpenAI 推廣的 agent instruction 格式 |
| apply_patch 格式 | 自訂 diff 格式，對 LLM 輸出更友善 |
| BM25 工具搜尋 | 大量 MCP tools 時的工具發現機制 |
| Rust 核心 | 高效能、記憶體安全的 agent 核心 |
| ChatGPT 帳號登入 | 不需 API key，Plus/Pro 方案直接使用 |
