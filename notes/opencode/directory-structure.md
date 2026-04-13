# OpenCode 目錄架構分析

> 基於 [sources/opencode](../../sources/opencode) 分析，版本：1.4.3

## 根目錄

```
opencode/
├── packages/          # Monorepo 各子套件
├── infra/             # SST v3 Cloudflare 部署設定
├── specs/             # API 規格文件
├── sdks/              # OpenAPI JSON spec（供 SDK 生成使用）
├── script/            # CI/CD 腳本
├── nix/               # Nix 開發環境 overlays
├── github/            # GitHub App 設定
├── patches/           # 套件 patch 檔
├── package.json       # Bun workspaces 入口
├── turbo.json         # Turborepo task 定義
├── sst.config.ts      # SST v3 部署設定（Cloudflare）
├── tsconfig.json      # 全域 TypeScript 設定
├── bunfig.toml        # Bun 設定
└── flake.nix          # Nix flake 開發環境
```

## packages/ 子套件樹狀圖

```
packages/
├── opencode/          # 核心 CLI 引擎（AI agent 邏輯）
├── app/               # Web UI（SolidJS）
├── ui/                # Design System 元件庫
├── sdk/
│   └── js/            # 公開 TypeScript SDK
├── desktop/           # Tauri 桌面應用（v1）
├── desktop-electron/  # Electron 桌面應用（v2）
├── console/
│   ├── app/           # SaaS 管理後台前端（SolidStart）
│   ├── core/          # 共用商業邏輯
│   ├── function/      # Cloudflare Workers function
│   ├── mail/          # 郵件範本
│   └── resource/      # 共用 resource 類型
├── enterprise/        # 企業版 Teams 功能
├── function/          # Cloudflare Worker API（GitHub App、Discord bot）
├── web/               # 文件站（Astro + Starlight）
├── plugin/            # Plugin SDK
├── script/            # 內部 build scripts
├── util/              # 共用工具函數（私有）
├── slack/             # Slack bot 整合
├── storybook/         # UI 元件開發工具
├── extensions/
│   └── zed/           # Zed editor extension
└── containers/        # Docker 建置腳本（base、bun-node、rust、tauri-linux）
```

## infra/ 基礎設施

SST v3 管理的 Cloudflare 部署架構：

| 檔案 | 部署目標 |
|------|---------|
| `infra/app.ts` | API Worker（含 Durable Object）、docs 文件站、web app |
| `infra/console.ts` | SaaS 管理後台 |
| `infra/enterprise.ts` | 企業版 Teams 服務 |
| `infra/secret.ts` | SST Secret 定義 |
| `infra/stage.ts` | 環境 domain 解析（production vs. staging） |

## specs/ 規格文件

| 檔案 | 內容 |
|------|------|
| `specs/project.md` | Project/Session REST API 規格 |
| `specs/v2/session.md` | Session API v2 設計說明 |

## packages/opencode/src/ 核心原始碼結構

```
src/
├── index.ts           # CLI 入口（yargs）
├── cli/               # CLI 命令定義
├── agent/             # Agent 定義與 prompt
│   └── prompt/        # Prompt 模板（txt）
├── session/           # 核心 session 邏輯
├── provider/          # LLM provider 整合
│   └── sdk/           # Provider 專用 SDK（如 copilot）
├── tool/              # Agent 可用工具（bash, read, write, edit...）
├── server/            # HTTP/SSE/WebSocket server（Hono）
│   └── routes/        # 各 domain 路由
├── bus/               # pub/sub 事件匯流排
├── sync/              # Event sourcing 機制
├── acp/               # Agent Client Protocol 實作
├── mcp/               # Model Context Protocol client
├── lsp/               # Language Server Protocol client
├── storage/           # SQLite 抽象（drizzle-orm）
├── snapshot/          # 檔案狀態 snapshot（git-based）
├── worktree/          # git worktree 管理
├── permission/        # Tool 呼叫 permission 評估
├── skill/             # SKILL.md 文件探索與載入
├── config/            # 設定檔讀取（JSONC）
├── auth/              # API key 與 OAuth 管理
├── project/           # Project 與 instance 抽象
├── git/               # git 操作封裝
├── pty/               # PTY 管理（供 bash tool 使用）
├── npm/               # npm plugin 動態安裝
├── effect/            # Effect library helpers
├── global/            # 全域路徑常數
├── flag/              # Feature flag 管理
├── env/               # 環境變數讀取
├── id/                # ULID-based ID 生成
├── share/             # Session 分享功能
├── control-plane/     # 雲端控制平面適配器
└── installation/      # CLI 版本與安裝路徑管理
```
