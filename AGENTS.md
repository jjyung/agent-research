# Agent Guide

## 語言規範

- 領域專有名詞（如 agent、submodule、LLM、prompt、tool call 等）維持英文原文
- 其餘一般描述、說明、筆記內容一律使用繁體中文

## Commit Message Format

```text
<type>: <description>

<optional body>
```

Types: feat, fix, refactor, docs, test, chore, perf, ci

## 新增研究專案流程

1. 以 git submodule 加入來源專案：

```bash
git submodule add https://github.com/<org>/<repo> sources/<repo>
mkdir -p notes/<repo>
```

2. 將該專案的 README 翻譯成繁體中文，存放於：

```
notes/<repo>/README.md
```

翻譯規範：

- 領域專有名詞維持英文原文，其餘翻譯為繁體中文
- 保留原始 Markdown 格式與程式碼區塊，僅翻譯說明文字
- 在檔案頂端標註來源，例如：`> 譯自 [原始 README](../../sources/<repo>/README.md)`
