> 基於 [sources/gemini-cli](../../sources/gemini-cli) 分析

# Gemini CLI 核心模組分析

## Agent Loop 設計

```
使用者輸入
  → AgentSession.sendStream()
  → GeminiChat（@google/genai SDK）
  → 收到 function_calls
  → Scheduler（policy 檢查 → 確認 → hook → 執行）
  → 注入 tool results → 下一 turn
  → finish_reason: stop 或使用者中斷
```

### Subagent 機制（`LocalAgentExecutor`）
- 每個 subagent 有獨立的 `ToolRegistry`
- 強制排除 `AgentTool`（防止無限遞迴）
- 強制注入 `complete_task` 作為唯一正常終止方法
- **Grace Period（1 min）**：超過 turn/time 限制時，給 agent 最後機會呼叫 `complete_task`

---

## 核心模組

### Agent 層（`packages/core/src/agent/`）

| 模組 | 職責 |
|------|------|
| `AgentSession` | 主 agent session；驅動 `sendStream()` 主迴圈 |
| `AgentProtocol` | agent 通訊協定定義 |
| `event-types.ts` | 所有 bus 事件型別 |

### Scheduler（`packages/core/src/scheduler/`）

協調 tool call 的生命週期：
1. **Policy 檢查**：`ApprovalMode` 決定是否需要確認
2. **Safety 檢查**：外部 safety checker（stdin/stdout 協定）
3. **Hook 呼叫**：`BeforeToolUse`
4. **執行工具**
5. **Hook 回調**：`AfterToolUse`

### Tools（`packages/core/src/tools/`）

**檔案操作**

| 工具 | 功能 |
|------|------|
| `read_file` | 讀取檔案（支援 offset/limit） |
| `edit` | 精確 string replace 編輯 |
| `write_file` | 寫入 / 建立檔案 |
| `glob` | Glob pattern 搜尋 |
| `grep` | 文字搜尋 |
| `ls` | 目錄列表 |

**Shell 與執行**

| 工具 | 功能 |
|------|------|
| `shell` | 執行 shell command |

**網路**

| 工具 | 功能 |
|------|------|
| `web_search` | Google Search grounding（附 sources metadata）|
| `web_fetch` | 抓取網頁內容 |

**Agent 控制**

| 工具 | 功能 |
|------|------|
| `agent` | 啟動 subagent（`LocalAgentExecutor`）|
| `complete_task` | Subagent 正常終止並回傳結果 |
| `enter_plan_mode` | 進入 Plan Mode（read-only 分析）|
| `exit_plan_mode` | 離開 Plan Mode，進入執行 |
| `ask_user` | 向使用者提問 |

**記憶體與任務**

| 工具 | 功能 |
|------|------|
| `memory` | 讀寫持久化 memory（global/project/extension）|
| `write_todos` | 更新 TODO 清單 |

**其他**

| 工具 | 功能 |
|------|------|
| `activate_skill` | 啟用 GEMINI.md skill 文件 |
| `update_topic` | 更新當前 conversation topic |

動態 MCP tools 透過 `packages/core/src/mcp/` 動態載入。

### Context 管理（`packages/core/src/context/`）

**`ChatCompressionService`**：
- 觸發條件：已用 token 達 context window 的 **50%**
- 壓縮策略：保留最新 **30%** 的對話歷史 + 產生摘要
- 壓縮後繼續對話，不中斷 session

### Safety 機制（`packages/core/src/safety/`）

支援外部 safety checker process（stdin/stdout 協定），可自訂拒絕邏輯。

### Hook 系統（`packages/core/src/hooks/`）

可攔截所有 tool 呼叫事件：
- `BeforeAgent`：agent 啟動前
- `BeforeToolUse`：工具執行前
- `AfterToolUse`：工具執行後

### Memory 系統（三層 GEMINI.md）

| 層級 | 路徑 | 範圍 |
|------|------|------|
| Global | `~/.gemini/GEMINI.md` | 所有 session |
| Project | `<project>/.gemini/GEMINI.md` | 特定 project |
| Extension | Extension 提供的 GEMINI.md | Extension 自訂 |

---

## Guardrail 機制

### ApprovalMode（四個等級）

| 模式 | 行為 |
|------|------|
| `PLAN` | 最嚴格；所有 tool calls 先進入計劃模式，使用者核准後才執行 |
| `DEFAULT` | 危險操作需確認 |
| `AUTO_EDIT` | 檔案編輯自動批准，其他需確認 |
| `YOLO` | 全部自動批准 |

### Path Validation
限制工具操作在 workspace 邊界內，防止逸出。

---

## 快取機制

### Prompt Cache
自動對 system prompt 注入 `cacheControl: ephemeral`（Gemini API 支援）。

### Context Compression
- 50% threshold 觸發壓縮，非等到 overflow 才處理
- 積極策略，比 opencode 的 compaction 更早啟動

---

## 特色功能

| 特色 | 說明 |
|------|------|
| Google Search Grounding | Gemini 原生 API，回應附帶 sources metadata |
| Plan Mode | read-only 分析 → 撰寫計劃 → 使用者核准 → 執行 |
| A2A Protocol | Agent-to-Agent 通訊協定支援 |
| VS Code IDE Companion | 可在 VS Code 側邊欄中整合 |
| Sandbox 隔離 | Docker / Podman 隔離執行環境 |
| Conversation Checkpointing | 儲存與恢復複雜 session |
| GitHub Action 整合 | 可在 PR review、issue triage 等 CI 流程中使用 |
