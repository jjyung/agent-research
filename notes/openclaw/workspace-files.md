# Workspace 提示檔案系統

> 分析日期：2026-04-10  
> 來源：[sources/openclaw/src/agents/workspace.ts](../../sources/openclaw/src/agents/workspace.ts)、[sources/openclaw/docs/reference/templates/](../../sources/openclaw/docs/reference/templates/)

---

## 概覽

OpenClaw 的 agent 行為由 **workspace 根目錄的純文字 Markdown 檔案**驅動，而不是 code 或設定 YAML。這些檔案在每次對話開始時注入 system prompt，讓「自定義 agent 行為」等同於「編輯文字檔案」。

預設 workspace 位置：`~/.openclaw/workspace/`（可用 `OPENCLAW_PROFILE` 環境變數切換 profile）

---

## 完整檔案清單

```
~/.openclaw/workspace/
├── AGENTS.md       ← 行為規範（最核心）
├── SOUL.md         ← 人格與核心身份
├── IDENTITY.md     ← 名字、外觀、emoji
├── USER.md         ← 關於使用者的個人資料
├── TOOLS.md        ← 本機環境注記（SSH、裝置名...）
├── BOOTSTRAP.md    ← 首次啟動儀式（用完即刪）
├── HEARTBEAT.md    ← 定時檢查任務清單
├── BOOT.md         ← 每次啟動執行的指令（選用）
├── MEMORY.md       ← 長期記憶（hand-curated）
├── memory/
│   └── YYYY-MM-DD.md  ← 每日流水記錄
└── skills/
    └── <skill>/
        └── SKILL.md   ← Skill 注入的工具說明
```

---

## 每個檔案的用途

### `AGENTS.md` — 行為規範（最重要）

**性質**：主 agent 的行為指引，內容最豐富  
**注入時機**：所有 session 的 system prompt 最前段（`CONTEXT_FILE_ORDER` 排序第 10）

預設模板涵蓋的行為規則：

| 區塊 | 內容 |
|------|------|
| Session Startup | 每次對話開始時要讀哪些檔案、按什麼順序 |
| Memory 機制 | 每日記錄 vs 長期記憶的區分；「不要憑記憶，要寫到檔案」 |
| Red Lines | 禁止行為（不洩露私人資料、不跑破壞性指令不問） |
| External vs Internal | 哪些動作可自由執行、哪些要先問 |
| Group Chat 規範 | 在群組裡何時說話、何時保持沉默 |
| Heartbeat 策略 | 定時輪詢的行為指引（vs Cron 的選擇邏輯） |
| Formatting 規範 | 各平台格式差異（Discord 不用表格、WhatsApp 不用標題） |

**更新機制**：agent 自己可以在 main session 中直接 edit 這份檔案，變更即時生效（下次 session 讀取）。

---

### `SOUL.md` — 人格核心

**性質**：定義 agent 的個性、價值觀、說話風格  
**注入時機**：system prompt，排序第 20

預設模板要求 agent：
- 有真實意見，而不是永遠附和
- 先自己試著解決，而不是立刻問問題
- 對外部行為保守、對內部探索大膽
- 意識到自己是「訪客」，使用者給了 access 就要對等尊重

**更新機制**：設計為 agent 與使用者共同演進的文件（`_This file is yours to evolve._`）。AGENTS.md 裡甚至要求 agent 更改 SOUL.md 時要告知使用者。

---

### `IDENTITY.md` — 身份標記

**性質**：存放 agent 的名字、形象、emoji（結構化的身份快照）  
**注入時機**：system prompt，排序第 30

```markdown
- Name: [你幫 agent 取的名字]
- Creature: AI assistant / ghost in the machine / ...
- Vibe: sharp / warm / chaotic / ...
- Emoji: 🦞
- Avatar: avatars/openclaw.png  ← workspace 相對路徑或 URL
```

**更新機制**：BOOTSTRAP.md 流程中由 agent 與使用者一起填寫；之後可自行編輯。

---

### `USER.md` — 使用者檔案

**性質**：agent 對使用者的了解（時區、稱呼、喜好、正在進行的項目）  
**注入時機**：system prompt，排序第 40

```markdown
- Name:
- What to call them:
- Timezone:
- Notes: 隨時間累積的個人化了解
```

**更新機制**：agent 在對話中學到新資訊時，應主動 update 這份檔案（「邊做邊學」）。

---

### `TOOLS.md` — 本機環境注記

**性質**：使用者環境特有的設定（與 skills 的通用指引分離）  
**注入時機**：system prompt，排序第 50

存放範例：
```markdown
### Cameras
- living-room → 客廳，180° 廣角
- front-door → 門口，動作觸發

### SSH
- home-server → 192.168.1.100, user: admin

### TTS
- Preferred voice: "Nova"
- Default speaker: Kitchen HomePod
```

**設計哲學**：Skills 是通用的（可分享），TOOLS.md 是私人的（你的基礎設施）。兩者分離讓你可以更新 skill 而不丟失本機配置。

---

### `BOOTSTRAP.md` — 首次啟動儀式（一次性）

**性質**：全新 workspace 的「出生儀式」腳本  
**生命週期**：存在 → agent 執行流程 → **自行刪除**

流程：
1. Agent 讀到這份檔案，知道這是全新 workspace
2. 與使用者對話，決定名字、性格、emoji
3. 填寫 IDENTITY.md 和 USER.md
4. 一起討論 SOUL.md
5. 引導連接 WhatsApp/Telegram 等 channel（可選）
6. **刪除 BOOTSTRAP.md**（任務完成）

**程式碼層面**：`ensureAgentWorkspace()` 偵測到全新 workspace 時會從模板生成此檔，設定 `bootstrapSeededAt` 時間戳記。當 BOOTSTRAP.md 被刪除且沒有 `setupCompletedAt` 時，標記 setup 完成。

---

### `HEARTBEAT.md` — 定時任務清單

**性質**：搭配 heartbeat cron 使用，定義每次「心跳」要執行什麼  
**使用方式**：預設為空（空檔案 = 直接回 `HEARTBEAT_OK`）；加入任務清單後 agent 才會執行

```markdown
# 範例 HEARTBEAT.md
- Check emails for urgent messages
- Check calendar for events in next 2h
```

**差異**：HEARTBEAT 是輕量定時批次（多項合一次 API call）；Cron 是精確時間點的獨立任務。

---

### `BOOT.md` — 啟動勾子（選用）

**性質**：Gateway 每次啟動時執行的指令（需啟用 `hooks.internal.enabled`）  
**使用情境**：開機問候、自動連線到某個服務、啟動前置作業

---

### `MEMORY.md` — 長期記憶（curated）

**性質**：agent 手動維護的長期記憶，相當於「人類的長期記憶」  
**安全限制**：**只在 main session 載入**，不在群組/頻道中注入（防止私人資訊外洩）  
**注入時機**：system prompt，排序第 70（最晚，避免污染群組 session）

**與 `memory/YYYY-MM-DD.md` 的區別**：

| | `MEMORY.md` | `memory/YYYY-MM-DD.md` |
|--|-------------|------------------------|
| 性質 | 精煉的長期記憶 | 每日流水記錄（raw log） |
| 更新者 | Agent 定期整理後更新 | Agent 即時記錄 |
| 載入條件 | 只有 main session | main session + 最近 2 天 |

---

## 注入順序與程式碼對應

```typescript
// src/agents/system-prompt.ts
const CONTEXT_FILE_ORDER = new Map([
  ["agents.md",    10],  // AGENTS.md
  ["soul.md",      20],  // SOUL.md
  ["identity.md",  30],  // IDENTITY.md
  ["user.md",      40],  // USER.md
  ["tools.md",     50],  // TOOLS.md
  ["bootstrap.md", 60],  // BOOTSTRAP.md（初始設定期）
  ["memory.md",    70],  // MEMORY.md（main session only）
])
```

Skills 的 SKILL.md 在 bootstrap section 之後作為獨立區塊注入。

---

## 初始化流程（`ensureAgentWorkspace()`）

```
第一次執行 openclaw
    │
    ▼
ensureAgentWorkspace({ ensureBootstrapFiles: true })
    │
    ├── 從 docs/reference/templates/ 讀取模板
    ├── writeFileIfMissing() × 6（不覆蓋已有內容）
    │   AGENTS.md / SOUL.md / TOOLS.md /
    │   IDENTITY.md / USER.md / HEARTBEAT.md
    │
    ├── 判斷是否為全新 workspace（無任何使用者內容）
    │   ├── 是 → 生成 BOOTSTRAP.md
    │   └── 否 → 標記 setupCompletedAt（skip bootstrap）
    │
    ├── 記錄 .openclaw/workspace-state.json
    │   bootstrapSeededAt / setupCompletedAt
    │
    └── ensureGitRepo()（workspace 預設是 git repo）
```

**重要**：所有模板用 `writeFileIfMissing()`（flag: `"wx"`），**已存在的檔案不會被覆蓋**，使用者的客製化內容永遠安全。

---

## .dev.md 變體

`docs/reference/templates/` 還有 `*.dev.md` 變體（`AGENTS.dev.md`、`SOUL.dev.md` 等），供開發模式的 workspace 使用，通常包含更多技術細節和 debug 指引。

---

## 設計哲學總結

1. **Prompt as filesystem** — agent 的記憶、人格、行為規則全部是普通文字檔，使用者可以用任何編輯器修改
2. **Agent as co-author** — agent 被明確授權（且被指示）去更新這些檔案，形成人機協作的記憶系統
3. **Privacy by file boundary** — 敏感的個人記憶（MEMORY.md）靠「不在群組 session 注入」來保護，而非加密
4. **Bootstrap as ritual** — 首次設定設計成一場對話儀式，而不是填表單，讓使用者與 agent 建立關係
5. **Separation of concerns** — 通用能力在 skills（SKILL.md），個人環境在 TOOLS.md，兩者不互相污染
