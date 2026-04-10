> 本文根據 [sources/openclaw/src/infra/heartbeat-runner.ts](../../sources/openclaw/src/infra/heartbeat-runner.ts)、[heartbeat-wake.ts](../../sources/openclaw/src/infra/heartbeat-wake.ts)、[auto-reply/heartbeat.ts](../../sources/openclaw/src/auto-reply/heartbeat.ts) 分析整理。

# openclaw Heartbeat 機制

## 什麼是 Heartbeat？

Heartbeat 是一種**定期主動觸發 agent 執行**的機制，不需要使用者發訊息。agent 在背景依排程讀取 `HEARTBEAT.md`，執行其中的指示，可主動向使用者發送通知。

---

## Timer 何時建立？

**不是 cron job，而是 Gateway 啟動時建立的 `setTimeout` 鏈。**

啟動流程：
```
Gateway 啟動
  └─ server-runtime-services.ts
       └─ startHeartbeatRunner({ cfg })
            ├─ 讀取 openclaw.json 中 agents.defaults.heartbeat 設定
            ├─ 計算每個 agent 的 intervalMs（預設 30 分鐘）
            └─ scheduleNext() ← 設定第一個 setTimeout
                 └─ 到期時呼叫 requestHeartbeatNow({ reason: "interval" })
                      └─ run() 執行後再次 scheduleNext()（重新排程）
```

每次執行完畢，`finally` 區塊都會呼叫 `scheduleNext()` 重新設定下一個 timer，形成鏈式排程。

---

## 設定格式（openclaw.json）

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "activeHours": {
          "start": "09:00",
          "end": "22:00",
          "timezone": "Asia/Taipei"
        },
        "model": "gpt-4o-mini",
        "session": "main",
        "isolatedSession": false,
        "prompt": "(自訂 heartbeat prompt)",
        "target": "telegram",
        "lightContext": false,
        "ackMaxChars": 300
      }
    }
  }
}
```

設定可以熱重載：Gateway 接收到設定更新時呼叫 `runner.updateConfig(cfg)`，不需重啟即可調整 interval。

---

## 觸發機制（4 種 reason）

| reason | 來源 | 說明 |
|--------|------|------|
| `interval` | `startHeartbeatRunner` 內的 `setTimeout` 鏈 | 最常見，定時到期自動觸發 |
| `wake` / `hook` | `requestHeartbeatNow()` 被外部呼叫 | webhook mapping 設定 `wakeMode: "now"` 時，或 `next-heartbeat` hook |
| `exec-event` | bash 指令執行完成時 | 執行完畢後自動觸發，告知 agent 結果 |
| `cron` | CronService 的 cron job 結束時 | server-cron.ts 的 cron 任務完成後通知 |

**注意**：`exec-event`、`cron`、`wake`、`hook` 會**繞過** HEARTBEAT.md 空檔案的跳過判斷，強制執行。只有 `interval` reason 會受空檔案影響。

---

## HEARTBEAT.md 的讀取與執行邏輯

`resolveHeartbeatPreflight()` 決定是否執行本次 heartbeat：

```
讀取 HEARTBEAT.md
├── 空檔案或只有 Markdown header / 空列表項目
│   └─ skipReason: "empty-heartbeat-file" → 直接跳過，不呼叫 API（省費用）
├── 找不到檔案 (ENOENT)
│   └─ 繼續執行（prompt 說「if it exists」，讓 model 自行決定）
├── 包含 structured tasks（tasks: 區塊）
│   ├── 計算 due tasks（依各自 interval 判斷）
│   ├── 有到期任務 → 建立批次 prompt 僅執行 due tasks
│   └── 無到期任務 → prompt: null → skipReason: "no-tasks-due"（跳過）
└── 一般文字內容
    └─ 直接注入為 heartbeat prompt
```

「空」的定義（`isHeartbeatContentEffectivelyEmpty()`）：
- 空白行
- Markdown header（`# Header`，`#` 後有空格）
- 空的 list item（`- [ ]`、`* ` 等）

---

## Structured Tasks 格式

HEARTBEAT.md 支援進階任務格式，各任務有不同的執行間隔：

```markdown
tasks:
  - name: check-email
    interval: 60m
    prompt: Check emails and summarize urgent items

  - name: daily-summary
    interval: 24h
    prompt: Provide a daily progress summary
```

- 每個 task 的最後執行時間記錄在 session 的 `heartbeatTaskState` 中
- `isTaskDue()` 依 `taskState[task.name]` 判斷是否到期
- 任務完成後 `updateTaskTimestamps()` 更新 `heartbeatTaskState`

---

## HEARTBEAT.md 的更新機制

| 更新方式 | 說明 |
|----------|------|
| **使用者手動編輯** | 主要更新方式，直接修改 `~/.openclaw/workspace/HEARTBEAT.md` |
| **agent 在 main session 自行更新** | `AGENTS.md` 明確授權 agent 可 edit HEARTBEAT.md，例如完成任務後更新狀態 |
| **自動建立** | `ensureAgentWorkspace()` 初始化時從模板建立空白 HEARTBEAT.md（`flag: "wx"`，不覆蓋已有內容） |

**對話過程中不會自動修改 HEARTBEAT.md**，除非 agent 的工具使用明確寫入檔案。

---

## isolatedSession 機制

設定 `isolatedSession: true` 時：
- heartbeat 使用獨立的 session key（`<base>:heartbeat`）
- 每次執行建立空白新 session（不帶歷史對話）
- 避免每次傳送 ~100K tokens 的完整對話歷史給 LLM
- 交付路由仍使用主 session 的 `lastChannel`、`lastTo` 決定送達對象

重複的 `:heartbeat` suffix 會被壓縮（`<base>:heartbeat:heartbeat` → `<base>:heartbeat`），防止 wake 重試時產生無限層疊。

---

## 跳過（skip）邏輯總覽

`runHeartbeatOnce()` 依序進行以下檢查，任一條件成立即跳過：

```
1. areHeartbeatsEnabled() === false         → reason: "disabled"
2. isHeartbeatEnabledForAgent() === false   → reason: "disabled"
3. resolveHeartbeatIntervalMs() === 0       → reason: "disabled"
4. !isWithinActiveHours()                  → reason: "quiet-hours"
5. queueSize (Main lane) > 0               → reason: "requests-in-flight"
6. preflight.skipReason 存在               → reason: "empty-heartbeat-file"
7. session lane busy                       → reason: "requests-in-flight"
8. prompt === null（無到期任務）            → reason: "no-tasks-due"
9. visibility 全 false                    → reason: "alerts-disabled"
10. 重複內容（24h 內完全相同）              → reason: "duplicate"
```

---

## 回覆過濾（HEARTBEAT_OK token）

若 agent 回應包含 `HEARTBEAT_OK` token（表示「無事發生」），heartbeat 不會轉發給使用者。這是核心的「省擾」機制：

- `stripHeartbeatToken()` 從回應中移除 `HEARTBEAT_OK`
- 移除後若文字為空且無媒體附件 → `shouldSkip: true`，不發送訊息
- 若 `visibility.showOk === true`，可選擇性發送靜默的 `HEARTBEAT_OK` 作為活躍指示

---

## 設計重點

| 設計決策 | 理由 |
|----------|------|
| `setTimeout` 鏈而非 `setInterval` | 每次執行後重新計算，避免 drift；支援動態調整 interval |
| 空 HEARTBEAT.md → 跳過（不呼叫 API） | 大幅降低不使用時的 API 費用 |
| 缺失 HEARTBEAT.md → 繼續執行 | 允許不依賴檔案的 heartbeat 設定 |
| `exec-event` / `cron` / `wake` 繞過空檔案檢查 | 這些觸發是明確意圖，不應被靜默取消 |
| Structured tasks 各自計時 | 細粒度控制不同頻率的任務，不需多個 agent |
| `isolatedSession` | 長期使用後 session 歷史不會影響 heartbeat 執行成本 |
| Priority queue（heartbeat-wake.ts） | 多個非同步觸發來源合併，250ms coalesce，防止重複執行 |
