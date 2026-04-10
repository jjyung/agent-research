# Agent Research

研究熱門 GitHub Agent 專案的架構與設計哲學，作為學習參考。

## 目錄結構

```
agent-research/
├── notes/      # 架構筆記、設計哲學分析（與 sources 1:1 對應）
└── sources/    # clone 的開源 agent 專案
```

## 使用方式

`sources/` 下的專案以 **git submodule** 管理，可追蹤上游原始提交記錄。

```bash
# 新增專案（加入 submodule）
git submodule add https://github.com/<org>/<repo> sources/<repo>
mkdir -p notes/<repo>

# clone 本 repo 時一併初始化所有 submodule
git clone --recurse-submodules <this-repo-url>

# 已 clone 但尚未初始化 submodule
git submodule update --init --recursive

# 更新所有 submodule 至上游最新
git submodule update --remote --merge
```

## 筆記結構建議

每個專案的 `notes/<repo>/` 可包含：

- `README.md` — 專案概覽、核心設計哲學
- `architecture.md` — 架構分析
- `patterns.md` — 值得學習的設計模式
- `observations.md` — 個人觀察與心得

## 追蹤的專案

| 專案 | 來源 | 重點 |
|------|------|------|
| [openclaw](notes/openclaw/README.md) | [openclaw/openclaw](https://github.com/openclaw/openclaw) | Local-first 個人 AI 助理；Gateway control plane；multi-channel；skills platform |
