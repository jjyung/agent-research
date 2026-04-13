# OpenCode Packages 概覽

> 基於 [sources/opencode/packages](../../sources/opencode/packages) 分析，monorepo 使用 Bun workspaces + Turborepo

## 套件一覽

| 套件名稱           | npm 名稱                        | 類型                  | 技術棧                                   |
| ------------------ | ------------------------------- | --------------------- | ---------------------------------------- |
| `opencode`         | `opencode-ai`                   | 核心 CLI              | Bun、Effect、Hono、Vercel AI SDK、SQLite |
| `app`              | `@opencode-ai/app`              | Web UI                | SolidJS、Vite、TailwindCSS 4             |
| `ui`               | `@opencode-ai/ui`               | Design System         | SolidJS、@kobalte/core、Shiki、KaTeX     |
| `sdk/js`           | `@opencode-ai/sdk`              | 公開 SDK              | TypeScript、openapi-ts 生成              |
| `desktop`          | `@opencode-ai/desktop`          | 桌面 App（v1）        | Tauri 2、SolidJS、Vite                   |
| `desktop-electron` | `@opencode-ai/desktop-electron` | 桌面 App（v2）        | Electron 40、electron-vite               |
| `console/app`      | `@opencode-ai/console-app`      | SaaS 後台             | SolidStart、Stripe、OpenAuth             |
| `enterprise`       | `@opencode-ai/enterprise`       | 企業版 Teams          | SolidStart、Hono、Cloudflare R2          |
| `function`         | `@opencode-ai/function`         | Cloudflare Worker     | Hono、Durable Objects、Octokit           |
| `web`              | `@opencode-ai/web`              | 文件站                | Astro 5、Starlight、Shiki                |
| `plugin`           | `@opencode-ai/plugin`           | Plugin SDK            | Zod、@opentui/core                       |
| `slack`            | `@opencode-ai/slack`            | Slack Bot             | @slack/bolt                              |
| `util`             | `@opencode-ai/util`             | 工具函數（私有）      | TypeScript、Zod                          |
| `script`           | `@opencode-ai/script`           | Build scripts（私有） | TypeScript                               |
| `storybook`        | `@opencode-ai/storybook`        | UI 開發工具           | Storybook 10                             |
| `extensions/zed`   | —                               | Zed editor extension  | —                                        |
| `containers`       | —                               | Docker 建置腳本       | Docker                                   |

---

## 各套件詳細說明

### `packages/opencode` — 核心 CLI 引擎

整個專案的核心。實作 AI agent 邏輯、session 管理、provider 整合、tool 執行引擎、HTTP/WebSocket server、CLI 入口點。

- **主要入口**：`src/index.ts`（yargs CLI）、`bin/opencode`
- **特殊設定**：conditional exports（`#db`、`#pty`）依 bun/node 環境切換實作
- **詳細分析**：見 [core-modules.md](core-modules.md)

### `packages/app` — Web UI

主要的 web 前端應用，嵌入 desktop 與 web app 中共用。

- **主要入口**：`src/index.ts`
- **依賴**：`@opencode-ai/sdk`、`@opencode-ai/ui`、`@opencode-ai/util`
- **測試**：Playwright e2e

### `packages/ui` — Design System

共用 UI 元件庫，提供 components、i18n、theme、icons、sounds、fonts。

- **named exports**：`./components`、`./hooks`、`./context`、`./pierre`（diff 相關 UI）
- **Shiki** 用於 syntax highlighting
- **KaTeX** 用於數學公式渲染
- **@pierre/diffs** 提供 diff view

### `packages/sdk/js` — 公開 TypeScript SDK

供外部整合使用，包含 v1 與 v2 client/server。

- **生成方式**：`@hey-api/openapi-ts` 從 `sdks/openapi.json` 生成 client
- **包含**：REST client、type definitions、cross-spawn utility

### `packages/desktop` — Tauri 桌面 App（v1）

以 Tauri（Rust + WebView）封裝的桌面版 app。

- **整合的 Tauri plugin**：clipboard、deep-link、notification、store、updater
- **UI**：重用 `@opencode-ai/app`

### `packages/desktop-electron` — Electron 桌面 App（v2）

與 Tauri 版平行存在的 Electron 版本（可能為新世代替代方案）。

- **特色**：`@lydell/node-pty` 用於 terminal；Effect 用於 service 管理
- **建置工具**：electron-vite、electron-builder

### `packages/console` — SaaS 管理後台

多個子套件組成的 SaaS 後台系統：

- **`console/app`**：主前端（SolidStart）；整合 Stripe 訂閱、chart.js、@openauthjs/openauth
- **`console/core`**：共用商業邏輯
- **`console/function`**：Cloudflare Workers function
- **`console/mail`**：郵件範本（@jsx-email/render）
- **`console/resource`**：共用 resource 類型

### `packages/enterprise` — 企業版 Teams

企業版 Teams 功能，部署於 Cloudflare。

- **技術棧**：SolidStart + Hono + hono-openapi + R2 storage
- **API**：hono-openapi 提供 OpenAPI schema

### `packages/function` — Cloudflare Worker API

後端 API worker，處理 GitHub App、Discord bot 等整合。

- **`SyncServer`**：Durable Object，實作即時同步
- **主要入口**：`src/api.ts`
- **JWT**：jose 用於 token 驗證

### `packages/web` — 文件站

以 Astro + Starlight 建置的文件站，部署於 Cloudflare Pages（`docs.opencode.ai`）。

- **亦含**：landing page、API proxy
- **Vercel AI SDK**：用於互動式示範

### `packages/plugin` — Plugin SDK

第三方 plugin 的開發套件，定義 tool 介面與 TUI 整合介面。

- **主要入口**：`src/tool.ts`（工具介面）、`src/tui.ts`（TUI 介面）
- **@opentui/core + @opentui/solid**：TUI rendering

### `packages/slack` — Slack Bot

Slack bot 整合，透過 @slack/bolt 接收事件、呼叫 opencode SDK。

### `packages/extensions/zed` — Zed Editor Extension

Zed editor 的 OpenCode 整合 extension。

### `packages/containers` — CI/CD 容器

各平台的 Docker 建置腳本：base、bun-node、rust、tauri-linux。

---

## 技術棧總結

### 語言與 Runtime

- **TypeScript**（全專案；全 ESM `"type": "module"`）
- **Bun**（主要 runtime 與 package manager）
- **Rust**（Tauri 桌面版）

### UI 框架

- **SolidJS**（app、desktop、console、enterprise）
- **Astro**（文件站）
- **Tauri 2**（desktop v1，Rust WebView）
- **Electron 40**（desktop v2）

### 後端框架

- **Hono**（HTTP API server）
- **SolidStart**（SSR，console 與 enterprise）

### AI / LLM

- **Vercel AI SDK**（`ai` package）：統一的 LLM streaming 介面
- **`@ai-sdk/*`**：各 provider 的 Vercel AI SDK adapters

### 函數式程式設計

- **Effect**（核心 service/layer 模式，型別安全 effect system）
- **Zod**（schema validation）

### 資料庫 / 持久化

- **SQLite**（bun-sqlite / better-sqlite3）
- **drizzle-orm**（ORM）
- **Cloudflare R2**（物件儲存，enterprise）
- **Cloudflare Durable Objects**（即時同步 server）

### 協定

- **ACP（Agent Client Protocol）**：JSON-RPC over stdio
- **MCP（Model Context Protocol）**：外部工具整合
- **LSP（Language Server Protocol）**：語言服務整合

### 部署

- **SST v3**（Infrastructure as Code）
- **Cloudflare Workers / Pages / R2 / Durable Objects**
- **Turborepo**（monorepo build orchestration）

### 測試

- **Bun test runner**（核心 opencode 套件）
- **Playwright**（e2e，app 套件）
